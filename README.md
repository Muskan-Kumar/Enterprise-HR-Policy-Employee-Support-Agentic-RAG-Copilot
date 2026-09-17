# Enterprise HR Policy Employee Support Agentic RAG Copilot

An enterprise HR policy and employee-support assistant built with FastAPI, LangGraph, and retrieval-augmented generation (RAG).

The copilot answers questions from a private company knowledge base first. When the private evidence is insufficient, it can fall back to Tavily web search and clearly identifies the response as external information. Every chat request returns the answer source, citations, and an execution trace.

> This project is a reference implementation. It is not legal, employment, or HR advice. Validate responses with your organization’s HR team before using them as policy.

## Features

- **Agentic question routing**: sends HR and policy questions through the private knowledge base, while handling greetings and casual questions directly.
- **Private knowledge retrieval**: embeds and retrieves company documents from Pinecone using Hugging Face sentence-transformer embeddings.
- **Evidence grading**: evaluates whether retrieved evidence is sufficient before generating an answer.
- **Web fallback**: uses Tavily Search when private evidence is weak, with an optional query-rewrite retry.
- **Grounded responses**: instructs the LLM to answer only from the selected evidence and avoid inventing policy details.
- **Citations and traceability**: exposes private knowledge-base sources, web citations, and LangGraph execution steps in the UI and API response.
- **Document ingestion**: supports PDF, DOCX, Markdown, and plain-text files through an admin-protected upload endpoint.
- **Audit logging**: stores question, timestamp, selected source, and workflow trace in a local SQLite database.
- **Web interface and API**: includes a browser chat UI plus JSON endpoints for health checks, chat, and ingestion.
- **Container-ready deployment**: includes a Dockerfile for running the FastAPI service on port `8080`.

## How It Works

```text
User question
		 |
		 v
Route: HR/policy or direct conversation?
		 |
		 +--> Direct response
		 |
		 +--> Retrieve private KB from Pinecone
							|
							v
				Grade private evidence
							|
							+--> Good: answer from private KB
							|
							+--> Weak: Tavily web search
														|
														v
											Grade web evidence
														|
														+--> Good: answer from web evidence
														+--> Weak: rewrite and retry, or return insufficient evidence
```

## Technology Stack

- **Backend**: Python, FastAPI, Uvicorn, Pydantic Settings
- **Agent workflow**: LangGraph, LangChain Core, LangChain Community
- **LLM**: Groq via `langchain-groq` (default model: `llama-3.3-70b-versatile`)
- **Private RAG**: Pinecone, `langchain-pinecone`, Hugging Face embeddings, `sentence-transformers/all-MiniLM-L6-v2`
- **Web search**: Tavily via `langchain-tavily`
- **Document processing**: PyPDF, python-docx, LangChain text splitters
- **Frontend**: Server-rendered Jinja2 template with vanilla HTML, CSS, and JavaScript
- **Audit storage**: SQLite
- **Deployment**: Docker

## Requirements

- Python `3.13+` for the project configuration, or Docker
- A Groq API key
- A Pinecone API key and index access
- A Tavily API key for web fallback

The default embedding model produces 384-dimensional vectors. The application creates the configured Pinecone index automatically if it does not already exist.

## Quick Start

### 1. Clone and create an environment

```bash
git clone https://github.com/<your-account>/<your-repository>.git
cd Enterprise-HR-Policy-Employee-Support-Agentic-RAG-Copilot

python -m venv .venv
# Windows PowerShell
.venv\Scripts\Activate.ps1
# macOS/Linux
# source .venv/bin/activate

pip install -r requirements.txt
```

### 2. Configure environment variables

Create a `.env` file in the repository root:

```dotenv
GROQ_API_KEY=your-groq-api-key
TAVILY_API_KEY=your-tavily-api-key
PINECONE_API_KEY=your-pinecone-api-key

# Optional settings
GROQ_MODEL=llama-3.3-70b-versatile
PINECONE_INDEX_NAME=fde-hr-policy-rag
PINECONE_NAMESPACE=company-hr-kb
EMBEDDING_MODEL=sentence-transformers/all-MiniLM-L6-v2
TOP_K=4
MAX_RETRIES=1
ADMIN_API_KEY=replace-this-in-production
```

Do not commit `.env`, API keys, uploaded documents, or generated audit data.

### 3. Index the sample knowledge base

The repository includes sample documents in `data/sample_kb/`.

```bash
python ingest_sample_kb.py
```

This loads supported documents, splits them into overlapping chunks, and writes their vectors to the configured Pinecone namespace.

### 4. Start the application

```bash
python run.py
```

Open [http://127.0.0.1:8080](http://127.0.0.1:8080) in a browser. The FastAPI OpenAPI documentation is available at [http://127.0.0.1:8080/docs](http://127.0.0.1:8080/docs).

## Docker

Build and run the service:

```bash
docker build -t hr-policy-copilot .
docker run --rm -p 8080:8080 --env-file .env hr-policy-copilot
```

The container listens on `0.0.0.0:8080`. Set `PORT` to use another container port.

## API

### Health check

```http
GET /api/health
```

### Ask a question

```http
POST /api/chat
Content-Type: application/json

{"question":"What is the process for requesting leave?"}
```

The response includes:

- `answer`: generated response
- `source_used`: `private_kb`, `web_search`, `direct`, or `insufficient_evidence`
- `citations`: private KB or web sources
- `trace`: workflow steps
- `rewritten_query`: the final retrieval query used by the workflow

### Ingest a document

```bash
curl -X POST http://127.0.0.1:8080/api/ingest \
	-H "X-Admin-Key: replace-this-in-production" \
	-F "file=@path/to/handbook.pdf"
```

Supported extensions are `.pdf`, `.docx`, `.md`, and `.txt`. The endpoint stores the uploaded file under `uploads/`, chunks it, and adds the vectors to Pinecone.

## Project Structure

```text
app/
	main.py                 FastAPI application and web UI setup
	api/routes.py           Health, chat, and document-ingestion endpoints
	core/config.py          Environment-backed application settings
	rag/state.py            LangGraph state and structured decisions
	rag/vectorstore.py      Embeddings and Pinecone integration
	rag/workflow.py         Routing, retrieval, grading, fallback, and generation
	services/ingestion.py   File loading and chunking
	services/audit.py       SQLite audit logging
data/sample_kb/           Example HR knowledge-base documents
static/                   CSS and browser JavaScript
templates/                Jinja2 HTML templates
ingest_sample_kb.py       Sample knowledge-base indexing script
run.py                    Local development entry point
Dockerfile                Container definition
```

## Security and Production Considerations

- Replace the default `ADMIN_API_KEY` before deployment and use a proper identity and access-management layer for production.
- Protect the chat and ingestion endpoints behind authentication, authorization, rate limiting, and HTTPS.
- Review uploaded files and apply storage, malware-scanning, retention, and size limits appropriate for your environment.
- Treat web fallback results as external information, not company policy. HR or legal review may be required.
- Move audit storage to a managed, access-controlled database for multi-instance deployments.
- Pin and regularly review dependency versions, API provider permissions, and Pinecone index configuration.
- Avoid sending confidential employee data to external LLM or web-search providers unless your organization has approved that data flow.

## License

See [LICENSE](LICENSE).