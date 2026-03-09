# cuda-base

Multi-architecture CUDA Docker base image for Phystack AI inference services.

[![Build and Deploy](https://github.com/phystack/cuda-base/actions/workflows/docker-build-deploy.yml/badge.svg)](https://github.com/phystack/cuda-base/actions/workflows/docker-build-deploy.yml)
[![Docker Hub](https://img.shields.io/docker/pulls/phygrid/cuda-base.svg?style=flat-square)](https://hub.docker.com/r/phygrid/cuda-base)

## Overview

cuda-base provides a production-ready Docker image with CUDA 12.9, TensorRT 10.12, cuDNN 9, and a CUDA-enabled FFmpeg build. It serves as the foundation for all Phystack AI services that require GPU-accelerated inference or video processing. The image supports both AMD64 (Intel/AMD) and ARM64 (NVIDIA Jetson) architectures.

## Tech Stack

| Layer | Technology |
|-------|------------|
| Base OS | Ubuntu 24.04 |
| CUDA | 12.9.0 runtime |
| TensorRT | 10.12.0.36 |
| cuDNN | 9 |
| FFmpeg | Custom build with NVENC/NVDEC/cuvid |
| Python | 3.12 (system default) |
| Web framework | FastAPI, Uvicorn |
| CI/CD | GitHub Actions (self-hosted runner) |
| Registry | Docker Hub (`phygrid/cuda-base`) |

## Prerequisites

- Docker with BuildKit support
- For GPU usage: NVIDIA Container Toolkit (`nvidia-docker2` or `--gpus` flag)
- For multi-arch builds: `docker buildx` with QEMU or native ARM64 builder

## Installation

```bash
docker pull phygrid/cuda-base:latest
```

Specific version:

```bash
docker pull phygrid/cuda-base:v1.0.51
```

Available architectures: `linux/amd64`, `linux/arm64`.

## Usage

### As a base image

```dockerfile
FROM phygrid/cuda-base:latest

COPY requirements.txt /app/
RUN pip install --no-cache-dir --break-system-packages -r requirements.txt

COPY . /app/
CMD ["python", "main.py"]
```

The image pre-installs FastAPI, Uvicorn, Pydantic, NumPy, Pillow, and Requests. Additional Python packages can be added via `pip install --break-system-packages`.

### Running with GPU access

```bash
# AMD64 host
docker run -d --gpus all -p 8000:8000 phygrid/cuda-base:latest

# NVIDIA Jetson (ARM64)
docker run -d --runtime nvidia --gpus all -p 8000:8000 phygrid/cuda-base:latest
```

### Pre-created directories

The image includes `/app/cache` owned by the non-root `appuser`. Downstream images should store model weights, data, and logs under `/app/`.

### Health check

```bash
docker run --rm phygrid/cuda-base:latest python /app/health_check.py
```

Verifies Python, CUDA, TensorRT, FFmpeg, and core Python packages.

## Project Structure

```
Dockerfile              # Multi-stage build (TensorRT, FFmpeg, PyAV, runtime)
Dockerfile.complex      # Alternate build variant
VERSION                 # Current semantic version (patch auto-incremented by CI)
DOCKER_DEPLOYMENT.md    # Detailed deployment and versioning docs
.github/workflows/
  docker-build-deploy.yml  # Build, tag, push to Docker Hub
```

## Environment Variables

These are baked into the image and apply to all downstream containers:

| Variable | Default | Description |
|----------|---------|-------------|
| `PYTHONUNBUFFERED` | `1` | Disable Python output buffering |
| `PYTHONDONTWRITEBYTECODE` | `1` | Prevent `.pyc` file creation |
| `PIP_NO_CACHE_DIR` | `1` | Disable pip cache |
| `PIP_DISABLE_PIP_VERSION_CHECK` | `1` | Skip pip version check |
| `CUDA_HOME` | `/usr/local/cuda` | CUDA toolkit root |
| `TRT_ROOT` | `/opt/tensorrt` | TensorRT installation root |

## Building from Source

```bash
git clone git@github.com:phystack/cuda-base.git
cd cuda-base
docker buildx build --platform linux/amd64,linux/arm64 -t phygrid/cuda-base:dev .
```

The build downloads TensorRT tarballs and FFmpeg source during the first run. On the self-hosted runner a persistent cache at `/opt/build-cache/cuda-base` avoids repeated downloads.

## Publishing

Automated via GitHub Actions on the `main` branch:

1. Push a change to `Dockerfile`, `VERSION`, or the workflow file.
2. CI reads the `VERSION` file. If the tag already exists, it auto-increments the patch number.
3. A multi-arch image is built and pushed to Docker Hub with both a versioned tag (`v1.0.51`) and `latest`.
4. CI creates a GitHub release and git tag.

To bump the minor or major version, edit the `VERSION` file manually before pushing. See [DOCKER_DEPLOYMENT.md](DOCKER_DEPLOYMENT.md) for full details.

## Related Documentation

- [DOCKER_DEPLOYMENT.md](DOCKER_DEPLOYMENT.md) -- versioning, triggers, and troubleshooting
- [Docker Hub: phygrid/cuda-base](https://hub.docker.com/r/phygrid/cuda-base)
- [LICENSE](LICENSE) -- MIT
