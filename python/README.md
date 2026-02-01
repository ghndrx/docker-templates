# Python Docker Templates

Production-ready Python Dockerfile templates with security best practices.

## Templates

| Template | Base Image | Use Case |
|----------|-----------|----------|
| `Dockerfile.uv` | python-slim | **Recommended** - Fast builds with UV package manager |
| `Dockerfile.pip` | python-slim | Traditional pip workflow |
| `Dockerfile.distroless` | distroless/python3 | Maximum security (no shell) |

## Features

All templates include:

- ✅ **Multi-stage builds** - Small final images (typically 50-150MB)
- ✅ **Non-root user** - Never run as root in production
- ✅ **Layer caching** - Dependencies cached separately from code
- ✅ **BuildKit caching** - Pip/UV cache persisted across builds
- ✅ **Tini init** - Proper signal handling and zombie reaping
- ✅ **Health checks** - Built-in health check endpoint
- ✅ **OCI labels** - Standard container metadata

## Quick Start

### Using UV (Recommended)

```bash
# Copy template
cp Dockerfile.uv ../my-project/Dockerfile

# Ensure you have pyproject.toml and optionally uv.lock
cd ../my-project

# Build
docker build -t myapp:latest .

# Run
docker run --rm -p 8000:8000 myapp:latest
```

### Using pip

```bash
cp Dockerfile.pip ../my-project/Dockerfile
# Ensure you have requirements.txt
docker build -t myapp:latest .
```

## Customization

### Change Python Version

```dockerfile
# In your Dockerfile or at build time
ARG PYTHON_VERSION=3.11
docker build --build-arg PYTHON_VERSION=3.11 -t myapp:latest .
```

### Add System Dependencies

If your app needs system libraries (e.g., for psycopg2, Pillow):

```dockerfile
# In the runtime stage, before USER appuser:
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        libpq5 \
        libjpeg62-turbo && \
    rm -rf /var/lib/apt/lists/*
```

### Custom Health Check

```dockerfile
# HTTP endpoint
HEALTHCHECK CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')"

# TCP port check
HEALTHCHECK CMD python -c "import socket; s=socket.socket(); s.connect(('localhost',8000)); s.close()"

# Custom script
HEALTHCHECK CMD python /app/healthcheck.py
```

### Entry Point Options

```dockerfile
# Gunicorn (production WSGI)
CMD ["gunicorn", "-w", "4", "-b", "0.0.0.0:8000", "app:app"]

# Uvicorn (production ASGI)
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]

# FastAPI with auto-reload (development only!)
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--reload"]
```

## Security Scanning

```bash
# Scan with Trivy
trivy image myapp:latest

# Scan with Grype
grype myapp:latest

# Generate SBOM
syft myapp:latest -o spdx-json > sbom.json
```

## Image Size Comparison

Typical sizes for a FastAPI app with common dependencies:

| Template | Approximate Size |
|----------|-----------------|
| Dockerfile.uv | ~120MB |
| Dockerfile.pip | ~130MB |
| Dockerfile.distroless | ~90MB |

## Best Practices

1. **Pin versions** - Use specific Python and dependency versions
2. **Use .dockerignore** - Exclude `.git`, `__pycache__`, `.venv`, tests
3. **Scan images** - Run Trivy/Grype in CI before deploying
4. **Use secrets properly** - Never bake secrets into images
5. **Multi-arch builds** - Use `docker buildx` for ARM64 support

## Example .dockerignore

```
.git
.gitignore
.dockerignore
Dockerfile*
docker-compose*.yml
*.md
*.pyc
__pycache__
.pytest_cache
.mypy_cache
.venv
.env
.env.*
tests/
docs/
```

## License

MIT
