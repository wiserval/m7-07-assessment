# ADR 0002 — Mount model artifacts at runtime over baking into the container image

**Date:** 2026-06-05
**Status:** Accepted
**Deciders:** ML Platform lead, MLOps lead

## Context

The Recommender Service loads a ~180 MB ONNX model artifact at startup. The
scenario requires continuous A/B testing against the model — multiple model
versions will be live simultaneously at any given time, and the product team
expects to launch new experiments frequently. The question is whether the model
artifact should be embedded in the container image at build time (bake) or
downloaded from S3 when the pod starts (mount).

## Decision

Mount the model artifact from S3 at pod startup. The artifact path is injected
as an environment variable:

    MODEL_S3_URI=s3://prod-models/ranker/v{semver}+{run-id}/model.onnx

The CI/CD deploy job reads this value from the MLflow registry's Production
stage and sets it in the EKS deployment manifest before rolling pods.

## Rationale

**A/B testing cadence.** With bake, launching a new experiment variant means
building a new container image, pushing to ECR, and executing a rolling
deployment — roughly 15 minutes end-to-end through the CI/CD pipeline. With
mount, a new variant is live after updating MODEL_S3_URI and restarting pods:
roughly 2 minutes. The scenario explicitly calls out continuous A/B testing;
coupling experiment iteration to the container build cycle is an unnecessary
bottleneck.

**Lifecycle separation.** Container image versions track code changes: API
contract updates, feature extraction logic, dependency bumps. Model artifact
versions track ML training runs: new data, hyperparameter tuning, architecture
experiments. These change at different cadences and are governed by different
processes (GitHub PRs vs MLflow promotion gates). Merging them into one artifact
means every model retrain triggers a code review process, and every code change
requires a model to be embedded. Neither is correct.

**Image layer efficiency.** A 180 MB artifact that changes on every training run
defeats Docker layer caching. Every rebuild uploads 180 MB to ECR. Keeping it
out of the image means the layers that change frequently (application code) are
small and cache correctly.

## Alternatives considered

**Bake the model into the container image.** Simpler startup — no S3 dependency,
no network call on init, easier local development (pull image and run). Rejected
because it ties model swap cadence to the CI/CD pipeline, which breaks the A/B
testing iteration speed requirement. It also creates an operational hazard: a
model promoted in MLflow has no effect on running pods until someone triggers a
new image build, which is not the expected behavior.

**Kubernetes init container for S3 download.** Functionally equivalent to the
chosen approach but splits the download into a separate container phase. Adds
manifest complexity (init container spec, shared volume) without meaningful
benefit. The Recommender Service's own startup code handles the S3 download
before the HTTP server binds, which achieves the same isolation with less YAML.

## Consequences

- Pod startup time increases by ~12 seconds for the S3 download over a VPC
  endpoint. Rolling deployments set `readinessProbe.initialDelaySeconds: 30`
  to account for model load time before traffic is routed.
- S3 availability is a pod startup dependency. If the model bucket is
  unreachable at init, the pod fails its readiness probe and stays out of
  rotation — existing replicas continue serving. This is the correct failure
  mode.
- MODEL_S3_URI must follow the format `ranker/v{semver}+{run-id}`. The
  deploy-model.yml pipeline validates this format before updating the
  deployment manifest. A path that does not match the scheme fails the
  pre-deploy check and the rollout does not proceed.
- Consistent with ADR 0001: because the model is mounted rather than baked,
  the container image tag scheme (`recommender:v{semver}-{sha}`) tracks the
  service code version, not the model version. Both are recorded in the
  deployment manifest for full traceability.