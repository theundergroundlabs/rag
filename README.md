# RAG Microservice

A production-ready Retrieval-Augmented Generation (RAG) microservice built with Python, FastAPI, and OpenAI GPT-4o. This service allows you to ingest PDF and CSV documents, create embeddings, and query your documents using natural language.

## Features

- 🔍 **Document Ingestion**: Support for PDF and CSV files
- 🧠 **Smart Chunking**: Intelligent text chunking with overlap for better context
- 🔢 **Vector Embeddings**: Uses OpenAI's `text-embedding-3-small` for high-quality embeddings
- 🗄️ **Vector Database**: Qdrant for efficient similarity search (easy to deploy with Docker)
- 🤖 **GPT-4o Integration**: Powered by OpenAI's latest language model
- 🔐 **API Authentication**: Secure API key-based authentication
- 🐳 **Docker Ready**: Fully containerized with Docker Compose, including a Qdrant service
- 📊 **RESTful API**: Clean and well-documented API endpoints

## Quick Start

### Prerequisites

- Python 3.11+
- Docker and Docker Compose
- OpenAI API key
- SWIG (for `faiss-cpu` if you were to switch back, but not needed for Qdrant setup. If you encountered issues with `faiss-cpu` compilation, ensure SWIG is installed: `brew install swig` on macOS)

### Installation

1. **Clone the repository**
   ```bash
   git clone <repository-url>
   cd rag-microservice
   ```

2. **Set up environment variables**
   ```bash
   cp env.example .env
   # Edit .env with your OpenAI API key, desired API key, and Qdrant settings if not using Docker defaults.
   ```

3. **Using Docker Compose (Recommended)**
   This is the easiest way to get started as it includes the Qdrant vector database.
   ```bash
   # Ensure Docker Desktop is running
   docker-compose up --build -d
   ```
   The API will be available at `http://localhost:8000`.
   Qdrant dashboard (optional) might be accessible depending on its configuration if you expose its UI port.

4. **Local Development (without Docker for the API, Qdrant can still be Dockerized or run in-memory)**

   a. **Set up a virtual environment** (Recommended)
       ```bash
       python3 -m venv venv
       source venv/bin/activate  # On Windows: venv\Scripts\activate
       ```

   b. **Install dependencies**
       If you encounter issues with `pydantic-core` on Python 3.12+ due to PyO3 versioning:
       ```bash
       PYO3_USE_ABI3_FORWARD_COMPATIBILITY=1 pip install -r requirements.txt
       ```
       Otherwise:
       ```bash
       pip install -r requirements.txt
       ```

   c. **Run Qdrant**
       - **Option 1: Using Docker (Recommended if not using docker-compose for everything)**
           ```bash
           docker run -d -p 6333:6333 -p 6334:6334 \
               -v $(pwd)/qdrant_storage:/qdrant/storage \
               qdrant/qdrant:v1.7.3
           ```
           Ensure your `.env` file has `QDRANT_HOST=localhost` and `QDRANT_IN_MEMORY=False`.
       - **Option 2: In-memory (for quick tests, data is not persisted)**
           Set `QDRANT_IN_MEMORY=True` in your `.env` file. No separate Qdrant instance needed.

   d. **Run the FastAPI service**
       ```bash
       # Ensure your .env file is configured (OPENAI_API_KEY, API_KEY, Qdrant settings)
       uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
       ```

## Configuration

Create a `.env` file in the root directory by copying `env.example`:

```bash
# Required
OPENAI_API_KEY=your_openai_api_key_here
API_KEY=your-secret-api-key

# Qdrant Configuration
QDRANT_HOST=localhost           # Host for Qdrant service (use 'qdrant-db' if using docker-compose for all services)
QDRANT_PORT=6333                # Port for Qdrant service
# QDRANT_API_KEY=your_qdrant_api_key # Optional: if Qdrant is secured with an API key
QDRANT_COLLECTION_NAME=rag_collection
QDRANT_IN_MEMORY=False          # Set to True to run Qdrant in-memory (for dev/testing, no persistence)

# Optional (with defaults for API)
CHUNK_SIZE=1000
CHUNK_OVERLAP=200
MAX_FILE_SIZE=10485760  # 10MB
UPLOAD_FOLDER=uploads
```

## API Documentation

### Authentication

All endpoints (except `/health` and `/`) require authentication via the `X-API-Key` header:

```bash
curl -H "X-API-Key: your-secret-api-key" http://localhost:8000/some_endpoint
```

### Endpoints

#### 1. Document Ingestion

**POST `/ingest`**

Upload and ingest PDF or CSV files into the vector database.

```bash
curl -X POST \
  -H "X-API-Key: your-secret-api-key" \
  -F "file=@document.pdf" \
  http://localhost:8000/ingest
```

**Response:**
```json
{
  "message": "Successfully ingested document.pdf",
  "filename": "document.pdf",
  "source_type": "pdf",
  "chunks_processed": 15,
  "details": { /* ... ingestion details ... */ }
}
```

#### 2. Query Documents

**POST `/query`**

Query the ingested documents using natural language.

```bash
curl -X POST \
  -H "X-API-Key: your-secret-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "What are the main benefits of the product?",
    "top_k": 5,
    "similarity_threshold": 0.7
  }' \
  http://localhost:8000/query
```

**Response:**
```json
{
  "answer": "Based on the provided context, the main benefits include...",
  "sources": [
    {
      "source": "document.pdf",
      "source_type": "pdf",
      "chunk_id": 5,
      "similarity_score": 0.89,
      "text_preview": "The main benefits of our product are..."
    }
  ],
  "query": "What are the main benefits of the product?",
  "context_used": 3
}
```

#### 3. Database Statistics

**GET `/stats`**

Get information about the Qdrant vector database collection.

```bash
curl -H "X-API-Key: your-secret-api-key" http://localhost:8000/stats
```

**Response (Example with Qdrant stats):**
```json
{
  "collection_name": "rag_collection",
  "status": "green",
  "optimizer_status": "ok",
  "total_vectors": 150,
  "total_indexed_vectors": 150,
  "vector_size": 1536,
  "distance_metric": "Cosine",
  "segments_count": 1,
  "error": null
}
```

#### 4. Health Check

**GET `/health`**

Check service health (no authentication required).

```bash
curl http://localhost:8000/health
```

## CLI Usage

Use the included CLI script to ingest documents directly. This script will use the Qdrant configuration from your `.env` file.

### Ingest a single file
```bash
# Ensure venv is active if not using Docker for this
python scripts/ingest.py path/to/document.pdf
```

### Ingest all files in a directory
```bash
python scripts/ingest.py path/to/documents/
```

## Docker Deployment

### Using Docker Compose (Recommended for full stack)

The `docker-compose.yml` file sets up:
- `rag-api`: The FastAPI application.
- `qdrant-db`: The Qdrant vector database service.

```bash
# Build and start all services
docker-compose up --build -d

# View logs for the API service
docker-compose logs -f rag-api

# View logs for Qdrant
docker-compose logs -f qdrant-db

# Stop and remove containers, networks, and volumes
docker-compose down -v # Use -v to also remove the qdrant_storage volume
```
The `rag-api` service is configured to connect to `qdrant-db` within the Docker network.

### Using Docker directly (for API service, assuming Qdrant is running elsewhere or in-memory)

```bash
# Build the API image
docker build -t rag-microservice-api .

# Run the API container
docker run -d \
  -p 8000:8000 \
  -e OPENAI_API_KEY=your_api_key \
  -e API_KEY=your_secret_key \
  -e QDRANT_HOST=your_qdrant_host_or_ip \
  -e QDRANT_PORT=6333 \
  # -e QDRANT_IN_MEMORY=True \ # If Qdrant host is not set
  -v $(pwd)/uploads:/app/uploads \
  --name rag-api-container \
  rag-microservice-api
```

## Architecture

The service is built with a modular architecture:

```
app/
├── main.py              # FastAPI application
├── core/
│   ├── config.py        # Configuration management (Pydantic Settings)
│   └── auth.py          # API Key Authentication
├── api/
│   └── routes.py        # API endpoints (FastAPI router)
├── services/
│   ├── embedding.py     # OpenAI embedding service
│   ├── vectordb.py      # Qdrant vector database client service
│   ├── ingestion.py     # Document processing and ingestion logic
│   └── rag.py           # RAG query processing logic
└── utils/
    └── chunking.py      # Text cleaning and chunking utilities

scripts/
└── ingest.py            # CLI for document ingestion

data/
└── vector_db/           # (No longer used by Qdrant if Qdrant is external or uses Docker volume)

uploads/                 # Temporary storage for uploaded files before processing
```

## Supported File Types

- **PDF**: Text extraction using `pdfplumber`
- **CSV**: Row-based processing with `pandas`

## Development

### Running Tests
(Test suite to be implemented)
```bash
# Install development dependencies (if any, e.g., pytest)
# pip install pytest pytest-asyncio httpx

# Run tests
# pytest
```

## Troubleshooting

### Common Issues

1. **OpenAI API Key Error**: Ensure `OPENAI_API_KEY` is correctly set in `.env`.
2. **Qdrant Connection Issues**:
   - If using Docker Compose, ensure the `qdrant-db` service is running (`docker-compose ps`).
   - If running Qdrant separately (Docker or native), verify `QDRANT_HOST` and `QDRANT_PORT` in `.env` are correct and Qdrant is accessible.
   - Check Qdrant logs for errors.
3. **`pydantic-core` build failure on Python 3.12+**:
   - Try installing with `PYO3_USE_ABI3_FORWARD_COMPATIBILITY=1 pip install -r requirements.txt`.
   - Consider using a slightly older Python version (e.g., 3.11) if issues persist, as it has wider support for compiled packages.
4. **File Upload Issues**: Check `MAX_FILE_SIZE` in config; ensure file types are supported.

### Logs

- **FastAPI App (Docker Compose)**: `docker-compose logs -f rag-api`
- **Qdrant (Docker Compose)**: `docker-compose logs -f qdrant-db`

## API Reference

Interactive API documentation is available when the service is running:
- Swagger UI: `http://localhost:8000/docs`
- ReDoc: `http://localhost:8000/redoc` # rag
