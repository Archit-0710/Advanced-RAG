# advRAG

An agentic Retrieval-Augmented Generation (RAG) API for answering questions over a private document collection. The service ingests PDF, DOCX, PPTX, HTML, and text files, stores their embeddings in Qdrant, applies local reranking, and generates answers through a LangGraph workflow.

## Architecture

```text
Client -> FastAPI -> NeMo Guardrails -> LangGraph planner
                                      -> Qdrant retrieval
                                      -> FlashRank reranking
                                      -> Portkey / Groq generation
                                      -> answer with source documents
```

Key components:

- **FastAPI** provides the HTTP API and interactive Swagger documentation.
- **NeMo Guardrails** screens incoming prompts before retrieval.
- **LangGraph** coordinates planning, retrieval, and response generation with per-thread memory.
- **Qdrant** stores and searches document embeddings.
- **Gemini embeddings** vectorize documents and search queries.
- **FlashRank** reranks retrieved passages locally.
- **Portkey and Groq** route LLM generation and provide fallback/cache configuration.

## Prerequisites

- Python 3.11 or newer
- A Qdrant cluster and API key
- A Gemini API key for embeddings
- A Groq API key for Guardrails
- A Portkey API key and configured provider slugs for generation
- [uv](https://docs.astral.sh/uv/) (recommended) or `pip`

## Setup

Clone the repository and create your local configuration:

```powershell
git clone <your-repository-url>
cd advRAG
Copy-Item .env.example .env
```

Fill in `.env` with your own credentials. Never commit this file.

```env
# Required for document embeddings
GEMINI_API_KEY=

# Required for vector search
QDRANT_CLUSTER_ENDPOINT=
QDRANT_API_KEY=

# Required by the Guardrails LLM
GROQ_API_KEY=

# Required for evaluation metric judging (can be a separate Groq key)
JUDGE_GROQ=

# Required for the Portkey-backed RAG LLM
PORTKEY_API_KEY=
GROQ_SLUG=
GROQ_SLUG_2=

# Optional observability
LOGFIRE_TOKEN=
```

`GROQ_SLUG` and `GROQ_SLUG_2` are Portkey provider/virtual-key slugs, not raw Groq API keys. If your Portkey workspace requires a saved gateway configuration, create one in the Portkey dashboard and use its `pc-...` configuration slug in the gateway client setup.

Install the dependencies with `uv`:

```powershell
uv sync
```

Or with a virtual environment and `pip`:

```powershell
py -3.11 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
```

## Ingest documents

Place documents in a local `DATA` directory. The directory is intentionally ignored by Git because source documents may be large or private.

```text
DATA/
├── true_data/
└── noisy_data/
```

Create or recreate the Qdrant collection and ingest the files:

```powershell
uv run python -m app.ingestion.processor DATA --wipe
```

Omit `--wipe` to add documents without deleting the existing collection:

```powershell
uv run python -m app.ingestion.processor DATA
```

Supported formats are PDF, DOCX, PPTX, HTML/HTM, and TXT. Processed JSON metadata is written to `processed_data/`, which is also ignored by Git.

## Run the API

```powershell
uv run uvicorn app.main:app --reload --port 8000
```

If the virtual environment is already active, this also works:

```powershell
python -m uvicorn app.main:app --reload --port 8000
```

Open these URLs once the server is running:

- API documentation: `http://127.0.0.1:8000/docs`
- Health/home response: `http://127.0.0.1:8000/`
- LangGraph diagram: `http://127.0.0.1:8000/graph`

## Run the Streamlit UI (optional)

Start the API first, then launch the local chat interface in another terminal:

```powershell
uv run streamlit run ui/app.py
```

The UI calls `http://localhost:8000` by default. To point it at a deployed backend, add `BACKEND_URL` to `.env`:

```env
BACKEND_URL=https://your-api.example.com
```

## Run the evaluation suite

The evaluation suite is a Streamlit dashboard for testing the live RAG pipeline and guardrails against the datasets in `evals/`. It uses the FastAPI `/query` endpoint, so start the API first in a separate terminal:

```powershell
uv run uvicorn app.main:app --reload --port 8000
```

Then start the evaluation dashboard:

```powershell
uv run streamlit run evals/app.py
```

If the virtual environment is active, run:

```powershell
streamlit run evals/app.py
```

The dashboard runs in three steps:

1. **Ground truth:** Review the RAG question/answer pairs and guardrails test cases from `evals/golden_dataset.json`.
2. **Live pipeline:** Send each RAG question and guardrails input to the running API. The results remain in the current Streamlit session.
3. **Evaluation metrics:** Score the collected responses with RAGAS and tool correctness.

The RAG evaluation reports Faithfulness, Answer Relevancy, Context Precision, Context Recall, Answer Correctness, and Tool Correctness. The guardrails evaluation reports true/false positives and negatives, precision, recall, and accuracy.

The LLM-based metric judge uses `JUDGE_GROQ`. Set it in `.env` before starting the dashboard; it may be separate from the production `GROQ_API_KEY` to avoid consuming the production key's quota. The metric suite processes samples sequentially with cooldowns and may take approximately 50 minutes. Generated evaluation results and local caches under `evals/results/`, `evals/reports/`, and `evals/.cache/` are ignored by Git.

## API usage

Send a question with an optional thread ID. Reusing a `thread_id` preserves conversation state for that session.

```powershell
Invoke-RestMethod -Method Post `
  -Uri http://127.0.0.1:8000/query `
  -ContentType 'application/json' `
  -Body '{"q":"How does the job autoscaling process work?","thread_id":"demo-user"}'
```

Example response shape:

```json
{
  "question": "How does the job autoscaling process work?",
  "answer": "...",
  "thought_process": ["Start", "..."],
  "status": "Response generated.",
  "sources": []
}
```

## Repository layout

```text
app/
├── agents/                 # LangGraph state, planner, retriever, responder
├── gateway/                # Portkey client and LLM configuration
├── guardrails/             # NeMo Guardrails rules and initialization
├── ingestion/              # Parsers, chunking, and Qdrant indexing
├── services/retrieval/     # Embeddings, Qdrant search, and reranking
├── config.py               # Environment-backed application settings
└── main.py                 # FastAPI application
evals/
├── app.py                  # Streamlit evaluation dashboard
├── golden_dataset.json     # RAG and guardrails evaluation cases
├── pipeline.py             # Live /query evaluation runner
├── metrics.py              # RAGAS and tool correctness metrics
└── guardrails_eval.py      # Guardrails classification metrics
```

## Security notes

- Keep API keys only in `.env` or your deployment secret manager.
- Do not commit documents under `DATA/` unless you have permission to publish them.
- Review generated `processed_data/` before sharing it; it can contain extracted source text.

## License

Add a license before distributing this project publicly.
