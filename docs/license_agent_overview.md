# License Fetch & Verification Agent — Overview

This document explains how the repository discovers and extracts license information for dataset sources during imports, the architecture of the agent(s), key code locations, and a concise Q&A for reviewers.

**Summary**
- **Purpose:** Detect the source URL and extract license metadata (license type, license URL, attribution, restrictions, confidence) for provenance verification during data imports.
- **Approach:** 1 browser-driven site visit (Groq Compound browser automation) to scrape full page(s), then multiple lightweight checklist workers (text-only) that analyze the scraped content with focused prompts (one job per worker).

**High-level Components**
- **Scraper (one browser visit):** implemented in [agents/scraper_agent.py](agents/scraper_agent.py) and the `scraper` node in [graph.py](graph.py). Uses `GroqCompoundClient` to run browser_automation-enabled queries and saves raw output to `data/raw`/`data/scraped`.
- **Worker agents (many checklist items):** implemented in [agents/worker_agent.py](agents/worker_agent.py) and the `workers` node in [graph.py](graph.py). Workers are text-only (no browsing) and use compact prompts from `/prompts`.
- **Core Groq tooling:** low-level client in [agents/groq_client.py](agents/groq_client.py) and browser automation helper in [src/utils/groq_browser_automation.py](src/utils/groq_browser_automation.py). The latter contains the license-specific helpers `find_source_url`, `extract_all_metadata` and `_parse_license_content`.
- **Checklist & prompts:** the checklist that controls which workers run is [checklist/checklist.txt](checklist/checklist.txt). License-focused prompts live at [prompts/worker_license_check.txt](prompts/worker_license_check.txt) and [prompts/worker_license_url.txt](prompts/worker_license_url.txt).
- **Orchestration & output:** pipeline orchestration is in [orchestrator.py](orchestrator.py) and [graph.py](graph.py). Aggregation into an import document is in [agents/import_doc_agent.py](agents/import_doc_agent.py) and the import-doc node in [graph.py](graph.py).
- **Config & storage:** file locations and API keys in [config.py](config.py). Scraped/raw/output files are stored under `data/raw`, `data/scraped`, and `data/output` respectively.

**Data Flow (step-by-step)**
1. User or pipeline calls `run_pipeline(url, ...)` in [orchestrator.py](orchestrator.py) / [graph.py](graph.py).
2. Scraper node uses `GroqCompoundClient.query()` (browser automation enabled) to visit the site and returns structured JSON or raw text; saved to `data/scraped`.
3. Workers are driven from `checklist/checklist.txt`. Each worker loads its prompt (from `prompts/`), receives the scraped content as context, and runs a compact text-only model via `GroqCompoundClient.query_text()`.
4. License-specific steps:
   - If you start from a source NAME, `GroqBrowserAutomation.find_source_url()` attempts to resolve the official website and any license/data URLs using browser automation and heuristics (JSON extraction, URL regexes).
   - If you start from a URL, `GroqBrowserAutomation.extract_all_metadata()` will visit multiple pages and return a structured report including a `LICENSE & TERMS` section.
   - `_parse_license_content()` applies regex heuristics to extract `license_url`, `license_type`, `attribution`, `restrictions` and `confidence` from the scraped content.
5. Worker results are collected and compiled into the import document and Croissant JSON-LD.

**Key Implementation Details**
- `GroqCompoundClient` (`agents/groq_client.py`) wraps the Groq SDK and enables `browser_automation`+`web_search` via `compound_custom` flags.
- `GroqBrowserAutomation.extract_with_automation()` (`src/utils/groq_browser_automation.py`) sends prompts with browser tools enabled and returns `content`, `executed_tools` and `raw_response` for debugging.
- `find_source_url()` tries to return a JSON payload with `url`, `license_url`, `data_url`. It falls back to regex-based URL extraction if JSON is not provided by the model.
- `_parse_license_content()` uses conservative, token-light regex parsing; it avoids claiming licenses when evidence is absent and filters out Creative Commons links in some flows to reduce false positives.
- The workers use carefully authored single-purpose prompts in `prompts/` (see `worker_license_check.txt`) so each worker returns one focused answer; this makes results easier to aggregate and audit.

**Where to look in code (quick links)**
- Scraper / pipeline: [graph.py](graph.py) and [orchestrator.py](orchestrator.py)
- Browser automation & parsing: [src/utils/groq_browser_automation.py](src/utils/groq_browser_automation.py)
- Groq client wrapper: [agents/groq_client.py](agents/groq_client.py)
- Worker wrapper: [agents/worker_agent.py](agents/worker_agent.py)
- Scraper wrapper: [agents/scraper_agent.py](agents/scraper_agent.py)
- Prompts / checklist: [prompts/worker_license_check.txt](prompts/worker_license_check.txt), [prompts/worker_license_url.txt](prompts/worker_license_url.txt), [checklist/checklist.txt](checklist/checklist.txt)
- Configuration: [config.py](config.py)

**Limitations & Notes**
- Heuristic-based: the license extraction relies on model-generated text and regex heuristics — it can return false negatives and occasionally noisy outputs. Always include a human-in-the-loop verification for high-risk imports.
- Confidence field: the model may output an explicit `confidence` token; otherwise downstream code treats output conservatively.
- CreativeCommons filter: some helper code filters or treats Creative Commons links specially; review `_is_valid_license_url()` if you see missing CC links.
- Rate limits & costs: full pipeline uses browser-enabled model for scraping (higher cost); workers use compact models for efficiency.

**How to run (example)**
Run the pipeline from a Python REPL or script:

```bash
python -c "from orchestrator import run_pipeline; run_pipeline('https://example.com/dataset')"
```

Or call `run_pipeline()` programmatically from your app, passing `progress_callback` for UI updates.

**Questions & Answers (for reviewers)**

Q: How does the agent *locate* the canonical source URL and license page?
A: Two approaches are supported: (1) `find_source_url()` (browser automation) resolves a source name to candidate URLs and returns `license_url` if found; (2) when given a URL, the scraper visits the site (`extract_all_metadata()`), follows pages as needed, and looks for license/terms links. Both use model-assisted browsing plus regex/JSON parsing.

Q: How is the actual license text/type extracted?
A: After pages are scraped, `_parse_license_content()` looks for explicit license lines, links, and common keywords. Worker prompts such as `worker_license_check.txt` ask the model to locate explicit evidence and output structured answers. The pipeline avoids guessing — workers are instructed to return "Not found" if evidence is missing.

Q: Where are the prompts and how can I tweak them?
A: All prompts live in the `prompts/` directory. Edit `prompts/worker_license_check.txt` to change the license-extraction instructions. The checklist (`checklist/checklist.txt`) controls which prompts run and in what order.

Q: How accurate is this? Can we fully automate license approvals?
A: Not recommended to fully automate approvals. The system is tuned to find explicit license statements reliably, but ambiguous sites or non-standard terms may require manual review. Use the `confidence` output and the quoted `Evidence` field from `worker_license_check.txt` to triage.

Q: How do I debug if a license isn't found?
A: Steps:
- Inspect `data/scraped/<safe_name>.json` and `data/raw/<safe_name>.txt` to see raw model output.
- Check the `raw_response` from `GroqBrowserAutomation` (returned in the agent result) while running locally.
- Re-run the scraper with a slightly modified prompt (see `prompts/scraper.txt`) or increase `max_retries`.

Q: How do we add new license heuristics (e.g., organization-specific DUA patterns)?
A: Update `_parse_license_content()` in [src/utils/groq_browser_automation.py](src/utils/groq_browser_automation.py) to include additional regexes and heuristics, and add tests/fixtures with representative pages.

Q: Where are results stored and how can I access them programmatically?
A: Pipeline writes artifacts to `data/raw`, `data/scraped` and `data/output`. The `run_pipeline()` return value includes `worker_results` (list of dicts). Use that return value for programmatic access.

Q: How does the system reduce false positives (claiming a license when none exists)?
A: Prompts explicitly instruct workers not to guess and to return "Not found" when no evidence exists. `_is_valid_license_url()` filters file-typed URLs and root paths. The aggregation steps prefer explicit evidence and a quoted `Evidence` field.

Q: Who should review the final decision?
A: A legal or data-governance reviewer should confirm approval for commercial reuse or redistribution. Use the pipeline to gather evidence and save reviewer time — not to replace legal sign-off.

---

If you'd like, I can:
- add a short runnable example notebook that runs the pipeline on a public dataset URL, or
- open a PR that logs `raw_response` objects for failed license extractions to a debug folder for easier triage.

Document generated from repository source files; see the linked files above for code-level detail.
