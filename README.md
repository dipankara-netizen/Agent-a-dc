# Agent-A-DC: Automated Dataset Catalog

Multi-agent metadata extraction pipeline that analyzes dataset URLs and generates structured documentation. Built with **LangGraph** for orchestration, **Groq Compound** for AI-powered scraping and analysis, and **Streamlit** for the web UI.

## How It Works

```
URL Input → Scraper (1 browser visit) → 22 Workers (text-only) → Output Documents
```

1. **Scraper Agent** — Visits the URL once using Groq Compound's browser automation, extracts all page content as structured JSON with `found: true/false` flags
2. **22 Worker Agents** — Each reads the stored JSON (no browser) and extracts one specific metadata field (license, dates, format, geo coverage, etc.)
3. **Output Agents** — Generate a Data Commons Import Document (`.docx`) and Croissant JSON-LD metadata (`.json`)

## Live Demo

Deployed on Streamlit Cloud: [Open App](https://dipankaratriya-cloud-agent-a-dc-app.streamlit.app)

## Features

- Single browser visit per URL (fast, avoids rate limits)
- 22 isolated worker agents — each has ONE job, preventing hallucination
- Data Commons Import Document generation (matching standard DC format)
- Croissant JSON-LD metadata generation (MLCommons standard)
- Dataset catalog with download buttons or step-by-step instructions
- Session state persistence — results survive tab switches and downloads
- Works locally (`.env`) and on Streamlit Cloud (Secrets)

## Project Structure

```
agent-a-dc/
├── app.py                    # Streamlit UI
├── graph.py                  # LangGraph pipeline (StateGraph)
├── orchestrator.py           # Thin wrapper for CLI usage
├── config.py                 # Paths, API keys, model config
│
├── agents/
│   ├── groq_client.py        # Groq API client (browser + text-only modes)
│   ├── scraper_agent.py      # Web scraper agent
│   ├── worker_agent.py       # Generic worker agent
│   ├── import_doc_agent.py   # Import document generator
│   └── croissant_agent.py    # Croissant metadata generator
│
├── prompts/                   # Editable prompt files (one per agent)
│   ├── scraper.txt            # Scraper instructions + JSON schema
│   ├── import_document.txt    # Import doc compilation prompt
│   ├── croissant.txt          # Croissant JSON-LD prompt
│   └── worker_*.txt           # 22 worker prompts (A1-A5, B6-B11, C12-C17, D1)
│
├── checklist/
│   └── checklist.txt          # Worker definitions: id | prompt_file | description
│
├── utils/
│   ├── docx_generator.py      # DC Import Doc format generator
│   ├── file_utils.py          # Checklist parser
│   └── prompt_loader.py       # Prompt file loader
│
├── schemas/
│   └── croissant_example.json # Reference Croissant schema
│
└── data/
    ├── raw/                   # Raw input data
    ├── scraped/               # Scraped JSON (one file per URL)
    └── output/                # Generated .docx, .json, summary
```

## Checklist Workers

| Section | ID | Description |
|---------|-----|------------|
| **A. Source Assessment** | A1 | Core Attributes (Place, Period, Variable, Values) |
| | A2 | Data Vertical (Education, Health, etc.) |
| | A3 | Geographic Level (Country, AA1, AA2) |
| | A4 | License Public & Permissible? |
| | A5 | License URL |
| **B. Format & Acquisition** | B6 | Parent/Provenance URL |
| | B6.1 | Child Source URLs (direct downloads) |
| | B7 | Source Format (CSV, API, XLS) |
| | B8 | Programmatic Access? |
| | B9 | Rate Limits? |
| | B10 | Sample Source URL |
| | B10.1 | Metadata Documentation URLs |
| | B11 | Download Steps |
| **B+ Dataset Catalog** | D1 | List all datasets with download info |
| **C. Availability** | C12 | Min Date |
| | C13 | Max Date |
| | C14 | Periodicity (Annual, Monthly, etc.) |
| | C14.1 | Date Resolution |
| | C14.2 | Place Resolution |
| | C15 | Refresh Frequency |
| | C16 | Last Refresh Date |
| | C17 | Next Expected Refresh |

## Setup

### Local

```bash
# Clone
git clone https://github.com/dipankaratriya-cloud/Agent-A-DC.git
cd Agent-A-DC

# Install dependencies
pip install -r requirements.txt

# Add API key
echo 'GROQ_API_KEY=gsk_your_key_here' > .env

# Run
streamlit run app.py
```

### Streamlit Cloud

1. Fork/deploy from this repo on [share.streamlit.io](https://share.streamlit.io)
2. Set `app.py` as the main file
3. Add secret in **Settings → Secrets**:
   ```toml
   GROQ_API_KEY = "gsk_your_key_here"
   ```

## Tech Stack

| Component | Technology |
|-----------|-----------|
| Orchestration | [LangGraph](https://github.com/langchain-ai/langgraph) |
| LLM / Scraping | [Groq Compound](https://groq.com) (browser automation + web search) |
| Web UI | [Streamlit](https://streamlit.io) |
| Document Generation | python-docx |
| Metadata Standard | [Croissant (MLCommons)](https://github.com/mlcommons/croissant) |

## Architecture

```
┌─────────────┐
│  Streamlit   │
│     UI       │
└──────┬───────┘
       │
┌──────▼───────┐
│  LangGraph   │
│  StateGraph  │
└──────┬───────┘
       │
┌──────▼───────┐     ┌──────────────┐
│   Scraper    │────▶│ data/scraped │
│ (1 browser)  │     │   *.json     │
└──────┬───────┘     └──────┬───────┘
       │                    │
┌──────▼────────────────────▼──┐
│    22 Workers (text-only)     │
│  A1-A5 | B6-B11 | C12-C17 | D1│
└──────────────┬───────────────┘
               │
       ┌───────┴───────┐
       │               │
┌──────▼──────┐ ┌──────▼──────┐
│ Import Doc  │ │  Croissant  │
│   (.docx)   │ │  (.json)    │
└─────────────┘ └─────────────┘
```

## License

MIT
