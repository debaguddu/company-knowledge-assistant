# Company Knowledge Assistant

A containerized retrieval-augmented generation (RAG) assistant for answering questions from company documents. The service ingests Markdown, PDF, DOCX, and TXT files, stores their embeddings in PostgreSQL with pgvector, reranks retrieved passages with Cohere, and generates grounded answers with OpenAI.

## Features

- FastAPI backend with a small browser UI
- Asynchronous document ingestion and ingestion-status polling
- PostgreSQL + pgvector for persistent vector search
- Redis semantic cache for repeated questions
- Cohere reranking of retrieved passages
- Source documents returned with every answer
- RAGAS evaluation script and sample question set

## Architecture

```text
Browser / API client
				|
				v
		 FastAPI
				|
				+--> PostgreSQL + pgvector  <--- document embeddings
				+--> Redis                  <--- semantic LLM cache
				+--> OpenAI                 <--- embeddings and answers
				+--> Cohere                 <--- passage reranking
```

## Requirements

- Docker Desktop with Docker Compose
- An OpenAI API key
- A Cohere API key
- A LangSmith API key is optional and only needed for tracing

The container uses Python 3.11. The `pyproject.toml` describes the package metadata, while the Docker image installs the runtime dependencies from `requirements.txt`.

## Configuration

Create a local `.env` file in the repository root. It is intentionally ignored by Git because it contains credentials:

```dotenv
DATABASE_URL=postgresql+asyncpg://postgres:postgres@postgres:5432/postgres
OPENAI_API_KEY=your-openai-api-key
CO_API_KEY=your-cohere-api-key
REDIS_URL=redis://cka-redis:6379/0

# Optional LangSmith tracing
LANGCHAIN_TRACING_V2=true
LANGSMITH_API_KEY=your-langsmith-api-key
LANGSMITH_PROJECT=Company Knowledge Base

DATA_DIR=data
RETRIEVAL_K=5
```

Do not commit `.env` or paste real credentials into source files. Rotate any credential that has been exposed outside the intended secret store.

## Run With Docker

Start PostgreSQL, Redis, and the FastAPI service:

```bash
docker compose up --build
```

Open the web UI at [http://localhost:8000](http://localhost:8000). The first run may take a while while the image installs document-processing dependencies.

The local service endpoints are:

| Method | Endpoint | Description |
| --- | --- | --- |
| `GET` | `/` | Browser UI |
| `POST` | `/ingest` | Start background ingestion |
| `GET` | `/ingest/status` | Check ingestion state and statistics |
| `POST` | `/ask` | Ask a question |

Start ingestion from another terminal after the containers are ready:

```bash
curl -X POST http://localhost:8000/ingest
curl http://localhost:8000/ingest/status
```

Ask a question:

```bash
curl -X POST http://localhost:8000/ask \
	-H "Content-Type: application/json" \
	-d '{"question":"What is the company policy on remote work?"}'
```

The response includes `answer`, `sources`, and the retrieved `contexts`.

## Add Documents

Put source files under `data/`. Ingestion recursively scans these directories and uses the first directory component as the document category:

- `data/announcements/`
- `data/faqs/`
- `data/guides/`
- `data/handbooks/`
- `data/policies/`

Supported file types are `.md`, `.pdf`, `.docx`, and `.txt`. After adding or changing documents, run ingestion again. The current `/ask` implementation requests the `guides` category, so guide documents are the active query collection until that filter is changed in `app/api.py`.

## Evaluate Retrieval Quality

Ensure the API is running and the database has been ingested, then run the evaluation script from the `app` directory so its default path resolves to `seed/qna_test.json`:

```bash
cd app
python eval_ragas.py
```

The script evaluates faithfulness, answer relevancy, context precision, and context recall using the sample questions in `seed/qna_test.json`.

## Project Layout

```text
app/
	api.py             FastAPI routes and ingestion lifecycle
	ingest.py          Document loading, chunking, and indexing
	rag.py             Retrieval, reranking, prompting, and generation
	eval_ragas.py      RAGAS evaluation runner
	static/            Browser UI
data/                Source documents
init-db/init.sql     pgvector schema initialization
seed/                Evaluation questions and answers
docker-compose.yml   PostgreSQL, Redis, and app services
Dockerfile           Application image definition
```

## Stop Services

```bash
docker compose down
```

To also remove the persisted PostgreSQL and Redis data volumes:

```bash
docker compose down -v
```
