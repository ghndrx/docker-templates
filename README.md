# Docker Templates

![Docker](https://img.shields.io/badge/Docker-24+-2496ED?style=flat&logo=docker&logoColor=white)
![License](https://img.shields.io/badge/License-MIT-blue)

Optimized Dockerfile templates with multi-stage builds, security scanning, and minimal attack surface.

## Templates

```
├── python/        # Python 3.11+ with UV/pip
├── node/          # Node.js with pnpm/yarn
├── go/            # Go with scratch final image
├── java/          # Java with Eclipse Temurin
├── rust/          # Rust with musl for static binaries
├── multi-stage/   # Advanced multi-stage patterns
└── security/      # Hardened base images
```

## Features

- ✅ Multi-stage builds (small final images)
- ✅ Non-root users
- ✅ Minimal base images (distroless, alpine, scratch)
- ✅ Layer caching optimization
- ✅ Security scanning with Trivy/Grype
- ✅ SBOM generation

## Usage

```bash
cp python/Dockerfile.template ./Dockerfile
# Customize for your app
docker build -t myapp .
```

## License

MIT
