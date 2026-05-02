# 🤖 DocBot — AI-Powered Document Assistant

> Chat with your documents, query databases, generate charts, and forecast trends — all in one place.

> **🌐 Live Demo:** https://yourdocbot.streamlit.app/ 

[![Python](https://img.shields.io/badge/Python-3.11+-blue?logo=python)](https://www.python.org/)
[![Streamlit](https://img.shields.io/badge/Streamlit-1.35+-red?logo=streamlit)](https://streamlit.io/)
[![LangChain](https://img.shields.io/badge/LangChain-0.2+-green)](https://langchain.com/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

---

## 📸 Screenshots

### Home Screen
![DocBot Home Screen](screenshots/screenshot_home.png)

### File Upload
![File Upload Interface](screenshots/screenshot_upload_file.png)

### Chat & RAG Pipeline
![Chat and RAG Pipeline](screenshots/screenshot_sql.png)

### SQL Database Connection
![SQL Database Connection](screenshots/screenshot_chat.png)

---

## ✨ Features

| Pipeline | Trigger | What it does |
|----------|---------|--------------|
| 🟢 **RAG** | Summarize / explain / Q&A | Retrieves relevant chunks from uploaded docs via FAISS vector search and answers with source citations |
| 📊 **GRAPH** | "plot", "chart", "visualize" | Generates Plotly Express charts from CSV/Excel data or extracts numbers from PDF/Word |
| 🔵 **DATA** | "show top 10", "filter", "sort" | Runs LLM-generated Pandas code on tabular files and returns an interactive table |
| 🔮 **FORECAST** | "predict", "forecast", "trend line" | Fits linear/polynomial regression or Prophet models and projects future values |
| 🗄️ **SQL** | "query the database", "SELECT …" | Translates natural language to SQL, runs it against your connected database, and summarizes results |

**Additional highlights**

- 🔄 **Auto-fallback** — if the primary model is rate-limited, DocBot automatically retries with the next available model
- 💬 **Conversation memory** — follow-up questions remember the last N turns
- 📄 **Multi-format ingestion** — PDF, Word (.docx), plain text, CSV, Excel (.xlsx/.xls)
- ⬇️ **CSV export** — download any query or forecast result
- 🗄️ **SQL integration** — PostgreSQL, MySQL, SQLite, MSSQL, Oracle
- 🔒 **Sandboxed code execution** — LLM-generated Python runs inside RestrictedPython; SQL is validated to SELECT-only before execution

---

## 🏗️ Technology Stack

| Layer | Technology |
|-------|-----------|
| UI | [Streamlit](https://streamlit.io/) |
| LLM orchestration | [LangChain](https://langchain.com/) |
| Vector store | [FAISS](https://github.com/facebookresearch/faiss) + `sentence-transformers` (`all-MiniLM-L6-v2`) |
| LLM providers | Google Gemini 2.5 Flash · GitHub Models (GPT-4.1 family) |
| Data / charts | Pandas · Plotly Express |
| Forecasting | scikit-learn (linear/polynomial regression) · Prophet (optional) |
| Database | SQLAlchemy + psycopg2 / pymysql / pyodbc |
| File parsing | pypdf · python-docx · openpyxl · xlrd |
| Security | RestrictedPython · sqlparse |

---

## ⚙️ Prerequisites

- Python **3.11+**
- pip / venv
- At least **one** LLM API key:
  - [Google AI Studio](https://aistudio.google.com/apikey) → `GEMINI_API_KEY`
  - [GitHub Personal Access Token](https://github.com/settings/tokens) → `GITHUB_TOKEN`

---

## 🚀 Local Setup

### 1 — Clone the repo

```bash
git clone https://github.com/your-username/docbot.git
cd docbot
```

### 2 — Create and activate a virtual environment

```bash
python -m venv venv

# macOS / Linux
source venv/bin/activate

# Windows
venv\Scripts\activate
```

### 3 — Install dependencies

```bash
pip install -r requirements.txt
```

> **Prophet (optional):** requires `pystan`. Install separately if needed:
> ```bash
> pip install prophet
> ```

### 4 — Configure environment variables

```bash
cp .env.example .env
# Open .env and fill in your API keys
```

### 5 — Run the app

```bash
streamlit run app.py
```

The app will open at **http://localhost:8501**.

---


## 📦 Adding New Dependencies

```bash
pip install <package>
pip freeze > requirements.txt
```

---

## 🛡️ Security

DocBot was audited and hardened before public release. Key protections:

**Safe code execution** — LLM-generated Python (DATA and GRAPH pipelines) is compiled and run inside [RestrictedPython](https://restrictedpython.readthedocs.io/), preventing sandbox escapes via Python's object model. Falls back to restricted builtins if the library is unavailable.

**SQL write protection** — The NL→SQL pipeline validates every generated query through two layers before execution: `sqlparse` checks the AST for `SELECT`-only statements, and a regex pass blocks dangerous keywords (`DROP`, `DELETE`, `INSERT`, `UPDATE`, `TRUNCATE`, etc.). Pair this with a read-only database user for defence-in-depth.

**XSS prevention** — LLM output interpolated into HTML is escaped with `html.escape()`. An `escape_llm_content()` helper is provided for any future HTML rendering.

**File size limits** — Uploads are validated at 20 MB across all parsers (`parse_pdf`, `parse_docx`, `parse_csv`, etc.) to prevent OOM crashes.

**Secret hygiene** — `.gitignore` covers `*.env` and `.env*`. Environment template (`env.example`) includes guidance for read-only DB users and email allowlisting for public deployments.

**Privacy notice** — Users are informed that uploaded files are transmitted to Google (Gemini) and GitHub (GPT) APIs before any file is processed.

See [SECURITY.md](SECURITY.md) for deployment best practices, a verification checklist, and self-hosted / Streamlit Cloud guides.

---

## 🤝 Contributing

1. Fork the repository.
2. Create a feature branch: `git checkout -b feat/my-feature`.
3. Commit your changes: `git commit -m 'feat: add my feature'`.
4. Push to your branch: `git push origin feat/my-feature`.
5. Open a Pull Request.

---

## 📄 License

MIT © 2026 — see [LICENSE](LICENSE) for details.
