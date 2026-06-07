# syntax=docker/dockerfile:1.6
#
# Recommender Service — Multi-stage Dockerfile
#
# The ONNX model artifact (~180 MB) is NOT baked into this image.
# It is downloaded from S3 by entrypoint.sh at pod startup using MODEL_S3_URI.
# See architecture/adr/0002-bake-vs-mount-model.md for the full rationale.
#
# Image tag scheme: recommender:v{semver}-{7-char-sha}  (no 'latest' tag ever)
# Build:  docker build -t recommender:v1.3.0-a3f1b9c .
# Run:    See container/README.md for required environment variables.

# ── Stage 1: builder ──────────────────────────────────────────────────────────
# Creates a virtual environment and installs all Python dependencies.
# gcc and libssl-dev stay here; they never appear in the runtime image.

FROM python:3.11-slim-bookworm AS builder

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1

WORKDIR /build

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        gcc \
        libssl-dev \
    && rm -rf /var/lib/apt/lists/*

RUN python -m venv /venv
ENV PATH="/venv/bin:$PATH"

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt


# ── Stage 2: runtime ──────────────────────────────────────────────────────────
# Copies only the virtual environment from the builder stage.
# No gcc, no build headers, no pip binary in the final image.

FROM python:3.11-slim-bookworm AS runtime

LABEL org.opencontainers.image.title="recommender-service" \
      org.opencontainers.image.description="Personalized product recommendation inference service" \
      org.opencontainers.image.source="https://github.com/org/recommender" \
      org.opencontainers.image.vendor="Retail Platform" \
      org.opencontainers.image.base.name="python:3.11-slim-bookworm"

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PYTHONFAULTHANDLER=1 \
    PATH="/venv/bin:$PATH" \
    MODEL_DIR="/app/model"

# libgomp1: required by ONNX Runtime for multi-threaded CPU inference.
# curl: used only by the HEALTHCHECK probe.
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        libgomp1 \
        curl \
    && rm -rf /var/lib/apt/lists/*

# Non-root application user — uid/gid 1001
RUN addgroup --system --gid 1001 appgroup \
    && adduser --system --uid 1001 --gid 1001 \
               --no-create-home --ingroup appgroup appuser

WORKDIR /app

# Copy virtual environment from builder
COPY --from=builder /venv /venv

# Copy application source and startup script
COPY --chown=appuser:appgroup src/     ./src/
COPY --chown=appuser:appgroup scripts/entrypoint.sh ./entrypoint.sh
RUN chmod +x ./entrypoint.sh

# Model directory: intentionally empty at build time.
# Populated at pod startup by entrypoint.sh (aws s3 cp $MODEL_S3_URI).
# If this directory contains a model file after a docker build, the build
# process has deviated from ADR 0002 and must be corrected.
RUN mkdir -p /app/model && chown appuser:appgroup /app/model

USER appuser

EXPOSE 8000

# Readiness via the /health endpoint.
# --start-period=40s accounts for S3 model download (~12 s) + ONNX load (~5 s)
# with margin. Pod does not receive traffic until the probe passes.
HEALTHCHECK --interval=15s --timeout=5s --start-period=40s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

# entrypoint.sh behaviour (in order):
#   1. Assert MODEL_S3_URI, MODEL_VERSION, REDIS_URL are set; exit 1 otherwise.
#   2. aws s3 cp "${MODEL_S3_URI}" /app/model/model.onnx
#   3. Verify /app/model/model.onnx exists and size > 0; exit 1 otherwise.
#   4. exec uvicorn src.main:app --host 0.0.0.0 --port 8000 --workers 2
ENTRYPOINT ["./entrypoint.sh"]