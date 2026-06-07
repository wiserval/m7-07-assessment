# Container Plan — Recommender Service

## Bake vs mount

The ONNX model artifact (~180 MB) is **not in this image**.

It is downloaded from S3 by `entrypoint.sh` on every pod startup, using the
`MODEL_S3_URI` environment variable injected by the CI/CD deploy job. The full
decision is in [ADR 0002](../architecture/adr/0002-bake-vs-mount-model.md).
Short version: the scenario requires continuous A/B testing. Baking the model
in ties model swap cadence to the container build pipeline (~15 minutes per
variant). With mount, swapping a model version takes ~2 minutes: update
`MODEL_S3_URI` in the EKS deployment manifest, roll the pods. Container image
versions track code changes; model registry versions track ML training runs.
They are not the same artifact and should not be released together.

## Base image: `python:3.11-slim-bookworm`

**Why slim over alpine?** ONNX Runtime's pre-built wheels link against glibc.
Alpine uses musl libc, which requires building ONNX Runtime from source or
using community-patched wheels — both add fragile CI steps for a ~30 MB saving
that is not worth it at our image size. slim gives us glibc without the full
Debian package set.

**Why bookworm (Debian 12)?** Current Debian stable. Security patches are
active. buster and bullseye are approaching or past EOL.

**Why 3.11 over 3.12?** ONNX Runtime 1.17 carries tested support for Python
3.11. 3.12 support was added in 1.18; update once 1.18 is confirmed stable
in the project's dependency matrix.

In production, pin both stages to the full image digest:
`python:3.11-slim-bookworm@sha256:<digest>`. Digest pinning is omitted here
for readability; the CI/CD pipeline resolves and pins the digest at build time.

## Multi-stage structure

| Stage | Base | Purpose |
|---|---|---|
| `builder` | python:3.11-slim-bookworm | Installs gcc, libssl-dev; creates `/venv`; installs all Python deps |
| `runtime` | python:3.11-slim-bookworm | Copies `/venv` from builder; adds libgomp1 + curl only; runs as uid 1001 |

Build tools never appear in the runtime image. The virtual environment copy
means `pip` itself is not present in the final container.

## Image size estimate

| Component | Size (uncompressed) |
|---|---|
| python:3.11-slim-bookworm base | ~45 MB |
| libgomp1 + curl (apt) | ~8 MB |
| Virtual environment (all Python deps) | ~140 MB |
| Application source (`src/`) | ~5 MB |
| **Total uncompressed** | **~198 MB** |
| **Total compressed (ECR layer)** | **~130 MB** |

The 180 MB model artifact is never stored in or transferred with the image.
The dominant dependency by size is ONNX Runtime (~60 MB of the venv). If
image size becomes a hard constraint, `onnxruntime` can be replaced with a
trimmed build at ~40 MB saving, at the cost of a custom wheel in CI.

## Environment variables

| Variable | Required | Description |
|---|---|---|
| `MODEL_S3_URI` | Yes | Full S3 path to the ONNX artifact. Format: `s3://prod-models/ranker/v{semver}+{run-id}/model.onnx`. Set by CI/CD deploy job from MLflow Production stage. |
| `MODEL_VERSION` | Yes | Full model registry version string (e.g. `ranker/v1.3.0+abc1234def56`). Emitted verbatim as the `X-Model-Version` response header on every request. |
| `REDIS_URL` | Yes | Feature store connection. Format: `redis://{host}:{port}/0` |
| `ANN_REDIS_URL` | Yes | ANN item index connection. May point to same Redis cluster, different DB index. |
| `LOG_LEVEL` | No | Uvicorn log level. Default: `info` |
| `WORKERS` | No | Uvicorn worker count. Default: `2`. Recommended ceiling: `(2 × vCPU) + 1 = 17` on c6i.2xlarge, but keep at 2–4 to avoid ONNX Runtime thread contention across workers. |

`MODEL_VERSION` is the value that appears in monitoring. The alert that
detects serving a non-Production model version queries the `X-Model-Version`
header, which is set directly from this variable. Keep it in sync with
`MODEL_S3_URI` — the deploy job sets both atomically from the MLflow registry.

## Key dependencies

| Package | Version | Purpose |
|---|---|---|
| `fastapi` | 0.111.x | HTTP framework |
| `uvicorn[standard]` | 0.29.x | ASGI server |
| `onnxruntime` | 1.17.x | ONNX inference on CPU |
| `redis[hiredis]` | 5.0.x | Feature store and ANN index client |
| `boto3` | 1.34.x | S3 model download in `entrypoint.sh` |
| `numpy` | 1.26.x | Array operations |
| `pydantic` | 2.7.x | Request / response validation |
| `prometheus-fastapi-instrumentator` | 7.0.x | `/metrics` scrape endpoint |
| `opentelemetry-sdk` | 1.24.x | Distributed tracing |

## Running locally

```bash
# Build
docker build -t recommender:dev .

# Run (requires AWS credentials with s3:GetObject on the prod-models bucket)
docker run \
  -e MODEL_S3_URI=s3://prod-models/ranker/v1.3.0+abc1234def56/model.onnx \
  -e MODEL_VERSION=ranker/v1.3.0+abc1234def56 \
  -e REDIS_URL=redis://localhost:6379/0 \
  -e ANN_REDIS_URL=redis://localhost:6379/1 \
  -p 8000:8000 \
  recommender:dev
```

For local development without S3 access, bind-mount a local model file and
set `MODEL_S3_URI=local` — `entrypoint.sh` skips the S3 download when the
value is `local` and expects the file at `/app/model/model.onnx` already.

```bash
docker run \
  -v $(pwd)/model.onnx:/app/model/model.onnx:ro \
  -e MODEL_S3_URI=local \
  -e MODEL_VERSION=ranker/v1.3.0+dev \
  -e REDIS_URL=redis://localhost:6379/0 \
  -e ANN_REDIS_URL=redis://localhost:6379/1 \
  -p 8000:8000 \
  recommender:dev
```

## Image tagging

CI/CD produces exactly one tag per build:
`recommender:v{semver}-{7-char-sha}` (e.g. `recommender:v1.3.0-a3f1b9c`).
No `latest` tag is ever pushed. ECR lifecycle policy deletes untagged images
after 14 days. The semver prefix matches the model registry version for the
same release, making the container-to-model mapping unambiguous in every
dashboard and runbook.