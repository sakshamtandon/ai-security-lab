# RAG PDF Assistant

A local Retrieval-Augmented Generation app for uploading PDFs, indexing them in Qdrant, and asking questions about the uploaded content.

The app uses:

- FastAPI as the Inngest function server
- Inngest for workflow/event orchestration
- Streamlit for the UI
- Qdrant as the vector database
- OpenRouter for embeddings and answer generation
- `baai/bge-m3` embeddings with 1024-dimensional vectors

## Project Structure

```text
.
├── main.py              # FastAPI app and Inngest functions
├── streamlit_app.py     # Streamlit upload/query UI
├── data_loader.py       # PDF loading, chunking, and embeddings
├── vector_db.py         # Qdrant collection, upsert, and search logic
├── custom_types.py      # Pydantic models for step inputs/outputs
├── pyproject.toml       # Project dependencies
└── README.md
```

## Requirements

- Python 3.10+
- `uv`
- Docker
- Inngest CLI
- OpenRouter API key

## Environment Variables

Create a `.env` file in the project root:

```bash
OPENROUTER_API_KEY=your_openrouter_api_key
```

Optional:

```bash
INNGEST_API_BASE=http://127.0.0.1:8288/v1
```

`INNGEST_API_BASE` is used by the Streamlit app to poll the local Inngest dev server for function output. The default is already `http://127.0.0.1:8288/v1`.

## Install Dependencies

```bash
uv sync
```

## Run Qdrant

Start Qdrant with Docker:

```bash
docker run -p 6333:6333 -p 6334:6334 \
  -v "$(pwd)/qdrant_storage:/qdrant/storage:z" \
  qdrant/qdrant
```

Qdrant will be available at:

```text
http://localhost:6333
```

The app writes vectors to the collection:

```text
docs_bge_m3
```

This collection uses 1024-dimensional vectors for the `baai/bge-m3` embedding model.

## Run the App

You need three terminals.

### Terminal 1: Start FastAPI

```bash
uv run uvicorn main:app --reload --port 8000
```

FastAPI will expose the Inngest endpoint at:

```text
http://127.0.0.1:8000/api/inngest
```

### Terminal 2: Start Inngest Dev Server

```bash
inngest dev -u http://127.0.0.1:8000/api/inngest
```

The Inngest dev UI usually runs at:

```text
http://127.0.0.1:8288
```

### Terminal 3: Start Streamlit

```bash
uv run streamlit run streamlit_app.py
```

Streamlit will open a local app in your browser.

## How to Use

1. Open the Streamlit app.
2. Upload a PDF.
3. The app sends a `rag/ingest_pdf` event to Inngest.
4. The Inngest function loads the PDF, chunks it, embeds the chunks, and upserts them into Qdrant.
5. Ask a question in the Streamlit UI.
6. The app sends a `rag/query_pdf_ai` event to Inngest.
7. The query function embeds the question, searches Qdrant, and asks an OpenRouter model to answer using the retrieved context.

## Inngest Events

### Ingest PDF

Event name:

```text
rag/ingest_pdf
```

Payload:

```json
{
  "pdf_path": "/absolute/path/to/file.pdf",
  "source_id": "file.pdf"
}
```

### Query PDF

Event name:

```text
rag/query_pdf_ai
```

Payload:

```json
{
  "question": "What caused the incident?",
  "top_k": 5
}
```

## Models

Embeddings:

```text
baai/bge-m3
```

Answer generation:

```text
openrouter/free
```

`openrouter/free` routes to an available free model on OpenRouter. For more predictable results, replace it in `main.py` with a specific OpenRouter model.

## Notes

- If you change the embedding model, make sure the Qdrant vector dimension matches the new model.
- Existing vectors from a different embedding dimension cannot be mixed in the same Qdrant collection.
- If query results are empty, make sure you have uploaded and ingested a PDF first.
- Keep FastAPI, Inngest dev server, Qdrant, and Streamlit running at the same time.
