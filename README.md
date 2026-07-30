# SOCPilot AI

**SOCPilot AI** is an AI-powered Security Operations Center (SOC) investigation assistant built to autonomously triage, enrich, and analyse security alerts. 

By leveraging **LangGraph**, **LangChain**, and **Retrieval-Augmented Generation (RAG)**, it behaves like a junior SOC analyst: it parses alerts, identifies Indicators of Compromise (IoCs), queries threat intelligence APIs, retrieves MITRE ATT&CK techniques, and synthesises all evidence into a professional investigation report.

## 🚀 Features

- **Stateful Workflows (LangGraph)**: Orchestrates a multi-node pipeline from alert ingestion to final report generation.
- **Dynamic Tool Routing**: Automatically decides which threat intelligence tools to query based on extracted IoCs (e.g., IPs trigger AbuseIPDB, file hashes trigger VirusTotal).
- **Retrieval-Augmented Generation (RAG)**: Uses a `MultiQueryRetriever` backed by ChromaDB and HuggingFace embeddings to inject relevant cybersecurity knowledge into the LLM's reasoning process.
- **Dual-Memory System**:
  - **Short-Term Memory**: `MemorySaver` provides continuity within a single investigation session (`thread_id`).
  - **Long-Term Memory**: Stores completed incidents in ChromaDB to retrieve semantically similar historical alerts in future investigations.
- **Deterministic & Fallback Capabilities**: 
  - Dual LLM/regex IoC extraction ensures no indicators are missed.
  - Fully functional offline/demo mode if external API keys (Groq, VirusTotal, AbuseIPDB) are not provided.
- **Structured Pydantic Outputs**: Guarantees type-safe generation of the final `SOCReport` (JSON and Markdown formats).

## 🏗️ Technology Stack

| Category | Technologies |
|----------|--------------|
| Programming Language | Python 3.11+ |
| AI Framework | LangChain |
| Agent Framework | LangGraph |
| LLM | Groq (Llama 3.3 / Mixtral / Gemma) |
| Vector Database | ChromaDB |
| Embeddings | HuggingFace all-MiniLM-L6-v2 |
| Validation | Pydantic |
| Templates | Jinja2 |
| Memory | LangGraph MemorySaver |
| Threat Intelligence | VirusTotal API |
| Reputation Service | AbuseIPDB API |
| Knowledge Base | MITRE ATT&CK |
| CVE Database | NVD |
| Logging | Python Logging |
| Environment | dotenv |
| Serialization | JSON |

## 🛠️ Architecture

```
User provides alert text
         │
         ▼
┌─────────────────────┐
│  ALERT INGEST NODE  │  ← LLM + Regex extracts IoCs
└──────────┬──────────┘
           ▼
┌──────────────────────────────────────────────┐
│            PARALLEL ENRICHMENT PHASE         │
│  ┌───────────────┐   ┌──────────────────┐    │
│  │  MEMORY NODE  │   │    RAG NODE      │    │
│  └───────┬───────┘   └────────┬─────────┘    │
└──────────┼────────────────────┼──────────────┘
           ▼                    ▼
┌──────────────────────────────────────────────┐
│           CONDITIONAL TOOL ROUTING           │
│  IP found?  ──────────► AbuseIPDB Node       │
│  Hash found? ─────────► VirusTotal Node      │
│  CVE found?  ─────────► CVE Lookup Node      │
│  Suspicious process? ─► MITRE / Sigma Nodes  │
└──────────────────────────────────────────────┘
           ▼
┌─────────────────────┐
│   REASONING NODE    │  ← LLM synthesises evidence → SOCReport
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│    REPORT NODE      │  ← Writes .md/.json, stores incident to ChromaDB
└─────────────────────┘
```

## ⚙️ Installation

1. Clone this repository.
2. Install dependencies:
   ```bash
   pip install -r requirements.txt
   ```
3. Configure environment variables:
   ```bash
   cp .env.example .env
   ```
   Open `.env` and add your API keys. **Groq** (`GROQ_API_KEY`) is highly recommended for the LLM reasoning phase.

4. Seed the RAG Knowledge Base (Run this once):
   ```bash
   python setup_rag.py
   ```

## 💻 Usage

Run the agent via the CLI, passing an alert inline or via a file.

**Investigate an inline alert:**
```bash
python main.py --alert "Suspicious PowerShell execution on HR-PC-21 by john. Source IP: 185.120.33.8. Command: powershell -enc SQBmAC..."
```

**Investigate an alert from a file:**
```bash
python main.py --file my_alert.txt
```

**Session Continuity (Short-Term Memory):**
If an investigation requires multiple turns, use the same `--thread-id`:
```bash
python main.py --thread-id inc-101 --alert "Initial alert..."
python main.py --thread-id inc-101 --alert "The user also downloaded a file with hash 44d88612fea8a8f36de82e1278abb02f"
```

## 📁 Output

Generated reports are saved in the `reports/` directory as both:
- `RPT-*.md`: Beautifully formatted Markdown report (rendered via Jinja2).
- `RPT-*.json`: Machine-readable structured JSON report.
