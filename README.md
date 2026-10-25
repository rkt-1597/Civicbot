# CivicBot — A Municipal Services Guide for Cambridge

[![Live Demo](https://img.shields.io/badge/Live%20Demo-Launch%20CivicBot-brightgreen)](https://civicbot-h4lj7cct3zy4gguejookek.streamlit.app/)

**CivicBot** is a university web-mining mini project. It crawls two public council websites, cleans and extracts facts from service pages, analyses website structure and topics, and answers citizen questions about bins, council tax, licences, housing, fees, and phone numbers. Its retrieval-augmented chatbot answers only from crawled pages and links to the official source.

**Live demo:** [CivicBot — Try the Application](https://civicbot-h4lj7cct3zy4gguejookek.streamlit.app/)

## Target Websites

| Website | Purpose |
|---|---|
| [Cambridge City Council](https://www.cambridge.gov.uk/) | Council tax, licensing, housing, and contact details |
| [Greater Cambridge Shared Waste Service](https://www.greatercambridgewaste.org/) | Bins and recycling. `cambridge.gov.uk/bins-and-recycling` redirects here. |

## Architecture

The pipeline has four stages: **collection → preprocessing → mining/analysis → reporting**.

### Data Processing and Chatbot Pipeline

```text
robots_check ─► run_crawl ─► clean ─► extract ─► cluster_pages ─► link_graph ─► deadline_timeline
  (Module 1)    (Module 1)  (Module 2) (Module 2)  (Module 3)     (Module 3)    (Module 3)
                                                           │
                                      chat.index ─► chat.ask / chat.evaluate (Module 4)
                                                           │
                         change_detect ─► make_snapshot ─► Streamlit app.py (Module 5)
```

### Deployment Architecture

The application is deployed using Streamlit Community Cloud, with the code hosted on GitHub.

```text
GitHub Repository: rkt-1597/Civicbot
        │
        │ main branch
        ▼
Streamlit Community Cloud
        │
        │ Installs requirements.txt
        ▼
      app.py
        │
        ├── Dashboard and visualisations
        ├── Demo snapshot data (data/snapshot/)
        ├── Hybrid retrieval (FAISS + BM25)
        └── Answer generation
              ├── Gemini API (if configured)
              ├── Ollama (if available)
              └── Retrieval-only fallback
        │
        ▼
Live CivicBot Application
https://civicbot-h4lj7cct3zy4gguejookek.streamlit.app/
```

The deployment uses the existing Streamlit application entry point, `app.py`, and the committed demo snapshot. Streamlit Community Cloud installs the dependencies specified in `requirements.txt` and runs the application in its hosted environment.

- **Deployment platform:** Streamlit Community Cloud
- **Source repository:** [rkt-1597/Civicbot](https://github.com/rkt-1597/Civicbot.git)
- **Application entry point:** `app.py`
- **Deployment branch:** `main`

The hosting setup does not, by itself, change the underlying crawling, preprocessing, analysis, or retrieval architecture.

## Setup

**Requirement:** Python 3.10 or newer.

### 1. Create and activate a virtual environment

```bash
python -m venv venv
source venv/bin/activate
```

On Windows PowerShell, activate it with:

```powershell
.\venv\Scripts\Activate.ps1
```

### 2. Install dependencies

```bash
pip install -r requirements.txt
python -m spacy download en_core_web_sm
```

### 3. Configure environment variables

Copy `.env.example` to `.env`, then edit `.env` with your settings.

On macOS/Linux:

```bash
cp .env.example .env
```

On Windows PowerShell:

```powershell
Copy-Item .env.example .env
```

Configure these variables in `.env`:

- **`GEMINI_API_KEY=`** — Paste a key from [Google AI Studio](https://aistudio.google.com/apikey). This is optional. Without it, CivicBot uses a local Ollama server if one is running (`ollama pull llama3.2`); otherwise it runs in retrieval-only mode, showing the best-matching passages and links. The application is designed to run without a key, and the sidebar shows which mode is active.
- **`CIVICBOT_CONTACT_EMAIL=`** — Your contact email, sent in the crawler's User-Agent so the site owner can contact you.

> **Security:** Never commit `.env`. It is listed in `.gitignore`, and the key is read from the environment.

## Run Order

Run the pipeline steps in order:

```bash
# 1. Check robots.txt for both domains; the crawler refuses to start without this check
python -m crawler.robots_check

# 2. Run a quick test crawl; omit --max-pages for the full 300-page crawl
python -m crawler.run_crawl --max-pages 20

# 3. Clean text and scrub personal data
python -m preprocess.clean

# 4. Extract deadlines, fees, phone numbers, hours, and schedules
python -m preprocess.extract

# 5. Run TF-IDF + K-Means topic clustering before link_graph
python -m analysis.cluster_pages

# 6. Calculate PageRank, hubs, communities, and link-cluster findings
python -m analysis.link_graph

# 7. Generate the upcoming-dates chart
python -m analysis.deadline_timeline

# 8. Chunk and embed documents; build the FAISS/BM25 index
python -m chat.index

# 9. Evaluate Recall@3 using eval_questions.json
python -m chat.evaluate

# 10. Compare against the previous snapshot ("What's new")
python -m crawler.change_detect

# 11. Save outputs to data/snapshot/ for the demo
python make_snapshot.py

# 12. Start the application
streamlit run app.py
```

Ask a question from the command line:

```bash
python -m chat.ask "What time do I put my bins out?"
```

## Demo Snapshot Mode (No Key, No Network)

The app's sidebar toggle **Demo snapshot mode** loads everything from `data/snapshot/` (committed, about 8 MB). This lets the demo run without an API key or network connection: the embedding model loads from the local cache, and answers run in retrieval-only mode. Run the pipeline once on a machine with internet access to download the embedding model before going offline.

## What Each Module Does

| Module | Files | Output |
|---|---|---|
| **1. Crawler** | `crawler/robots_check.py`, `settings.py`, `civic_spider.py`, `run_crawl.py`, `change_detect.py` | `data/raw/pages.jsonl`, `links.csv`, `documents.csv`, `crawl_report.json`; `data/outputs/updates.json` |
| **2. Preprocessing** | `preprocess/clean.py`, `extract.py` | `data/clean/pages_clean.jsonl`, `entities.csv`, `page_entities.csv` |
| **3. Analysis** | `analysis/cluster_pages.py`, `link_graph.py`, `deadline_timeline.py`, `style.py` | Charts and reports in `data/outputs/` |
| **4. Chatbot** | `chat/index.py`, `ask.py`, `evaluate.py`, `eval_questions.json` | `data/outputs/index/`, `eval_report.json`, `eval_recall.png` |
| **5. App** | `app.py`, `make_snapshot.py` | Streamlit app and `data/snapshot/` |

All settings—including target URLs, seeds, exclusions, limits, and LLM models—are in `config.py` and can be overridden through `.env`.

## Results

*Results from the crawl of 25 October 2026.*

- **Crawl:** 300 pages (201 council, 99 waste service), 0 blocked by `robots.txt`, 9 broken links logged, 254 document links recorded (229 PDFs), and 9.5 minutes at 2 seconds per request per domain.
- **Cleaning:** 320,278 visible words → 159,130 clean words, after removing menus, banners, and duplicate lines; 48 names, 10 personal email addresses, and 3 mobile numbers scrubbed.
- **Extraction:** 730 entities: 69 deadlines, 157 dates, 405 fees, 48 phone numbers, 9 office-hour statements, and 42 collection-schedule rules.
- **Topics:** `k = 4` by mean silhouette: Waste & recycling, Taxi licensing, Council tax + Licences, and Housing. Eighty-seven thin hub pages were not clustered. Silhouette is low (about 0.07) for every `k`; the site's text does not separate into sharp topics.
- **Link-structure finding:** 81.9% of linked content-page pairs share a topic cluster once site menus are removed, compared with 61.4% when menus are included.
- **Chatbot retrieval:** Recall@3 90% (27/30), Recall@1 77%, MRR 0.83; all 8 off-topic questions were correctly refused at the 0.50 confidence threshold.

## Limitations

### Coverage

1. **No actual bin days.** Collection days come from a postcode lookup form, which is deliberately not automated. CivicBot can give the rules (for example, “bins out by 6am”) and link to the lookup, but cannot answer “your bin day is Tuesday”.
2. **300-page cap.** Many service pages were never reached, including planning, parking details, most news, and 87 pages that are mostly link lists. A question about an uncrawled page may receive “couldn't find that” or a near-miss page.
3. **No birth/death certificate content.** In England, registration is a county council service, so it is not covered by these sites.
4. **PDFs are only recorded.** The 229 PDF links were recorded but none were downloaded or read, so fee tables or forms that exist only in PDFs are invisible to the chatbot.
5. **One-off snapshot.** Change detection has only one baseline so far. The “What's new” tab shows actual changes only after the next crawl.

### Extraction and Cleaning

6. **Rule-based extraction.** Some “fees” are not citizen fees (for example, budget figures or “for every £1 you pay” on a jobs page), and office hours were found on only 6 pages.
7. **Most deadlines have no exact date.** Only 16 of 69 have a normalised date; the rest are relative (such as “within 21 days”) or have no year, and are left null on purpose.
8. **Imperfect name scrubbing.** Some brands are removed as if they were people (for example, “John Lewis”), while an unusual name written in lowercase or appearing as an ordinary word elsewhere on the site could be missed.

### Analysis

9. **Weak topic structure.** Silhouette is about 0.06–0.07 for every `k` from 3 to 8 (within seed noise), so the chosen `k = 4` is not strongly supported, and one cluster mixes council tax with licences. Cluster names come from a hand-written keyword list.
10. **Link finding depends on a threshold.** “Menu links” are defined as targets linked from at least 25% of pages; another threshold would change the 81.9% and 61.4% figures somewhat.

### Chatbot

11. **Small, self-written test set.** The 30 evaluation questions were written after seeing the crawled pages, so Recall@3 = 90% is probably optimistic for real users.
12. **Narrow confidence margin.** Off-topic questions scored up to 0.47 and in-scope ones as low as 0.54. Some real questions will be wrongly refused, and some off-topic ones answered.
13. **LLM modes are untested.** Gemini and Ollama were not tested on the development machine (no API key and no Ollama); only retrieval-only mode was exercised end to end. The figure guard only checks fees, UK phone numbers, and “day Month year” dates, not every kind of fact.
14. **English only.** Every answer is limited to the date of the crawl.

## Ethics and Compliance

See [`docs/ethics.md`](docs/ethics.md) for `robots.txt` and terms-of-use checks for both domains, rate limits, excluded page types, personal-data scrubbing, and how the app attributes and links to original pages.

## Demo

- **[Launch CivicBot Live](https://civicbot-h4lj7cct3zy4gguejookek.streamlit.app/)**
- See [`DEMO_SCRIPT.md`](DEMO_SCRIPT.md) for a five-minute walkthrough.
