# cicd-pipeline-demo

Complete CI/CD pipeline for a containerized API. On every push: lint, tests, Docker image build, and automatic publish to GitHub Container Registry. Demonstrates end-to-end continuous integration and delivery.

## Stack

- **API**: FastAPI + Uvicorn (Python 3.12)
- **Lint**: Ruff
- **Tests**: Pytest + HTTPX
- **Container**: Docker / Docker Compose
- **CI/CD**: GitHub Actions → GitHub Container Registry (GHCR)

## Pipeline

```
push → lint (ruff) → test (pytest) → build & push (ghcr.io)
```

PRs only run lint + test. Image publishing only happens on pushes to `main`/`master`.

## Endpoints

| Method | Path            | Description            |
|--------|-----------------|------------------------|
| GET    | `/`             | Greeting + status      |
| GET    | `/health`       | Health check           |
| GET    | `/items/{id}`   | Item detail            |

## Local usage

```bash
# With Docker Compose
docker compose up --build

# Without Docker
pip install -r requirements.txt
uvicorn app.main:app --reload
```

The API is available at `http://localhost:8000`.  
Auto-generated docs at `http://localhost:8000/docs`.

## Tests

```bash
pip install -r requirements.txt
cd app
pytest test_main.py -v
```

## Published image

Every push to `main` automatically publishes to:

```
ghcr.io/<owner>/cicd-pipeline-demo:latest
ghcr.io/<owner>/cicd-pipeline-demo:sha-<commit>
```

To use the published image:

```bash
docker pull ghcr.io/<owner>/cicd-pipeline-demo:latest
docker run -p 8000:8000 ghcr.io/<owner>/cicd-pipeline-demo:latest
```
