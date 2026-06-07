# Capacity Plan — Recommender Service (Scenario X)

## Inputs

All numbers pulled from the locked assumptions table.

| Parameter | Value |
|---|---|
| Sustained RPS | 350 |
| Peak RPS | 800 |
| Sizing target (peak + 25% headroom) | 1,000 RPS |
| p95 latency SLO (internal) | 100 ms |
| p95 latency SLA (scenario) | 120 ms |
| Availability SLO | 99.9% |
| Model serving | Mount from S3 at pod init (ADR 0002) |

## Latency budget

The 120 ms end-to-end SLA breaks down as follows. The internal SLO target
of 100 ms leaves 20 ms of margin against the SLA.

| Segment | Allocated | Notes |
|---|---|---|
| ALB + TLS termination | 5 ms | Measured at P99 in staging |
| Redis feature lookup | 8 ms | ElastiCache r6g.large, sub-ms P50, 8 ms P99 budget |
| ANN retrieval (Redis index) | 5 ms | 200 candidates from 500k-item index |
| ONNX MLP ranker (CPU) | 22 ms | INT8 quantized, batch=1, c6i.2xlarge |
| Response serialization + egress | 10 ms | JSON encoding of top-N items |
| **Total (P50 estimate)** | **50 ms** | |
| **Margin to 100 ms SLO** | **50 ms** | |
| **Margin to 120 ms SLA** | **70 ms** | |

The cold-start path (popularity fallback, no ranker) takes approximately 3 ms
for a Redis lookup against a pre-computed popularity list.

## Replica sizing

**Per-request CPU time:** 22 ms (ranker inference) + ~5 ms (serialization) = 27 ms.
Redis lookups are async I/O and do not block the CPU during the wait.

```
CPU cores needed at 100% utilisation for 1,000 RPS:
  1,000 req/s × 0.027 s/req = 27 cores

At 60% target utilisation (safety margin):
  27 / 0.60 = 45 vCPUs required

c6i.2xlarge = 8 vCPU:
  ceil(45 / 8) = 6 replicas → 48 vCPUs total

Actual utilisation at sizing target (1,000 RPS):
  (1,000 × 0.027) / 48 = 56.3% — within the 60% ceiling

Actual utilisation at peak (800 RPS):
  (800 × 0.027) / 48 = 45% — comfortable headroom

Actual utilisation at sustained (350 RPS):
  (350 × 0.027) / 48 = 19.7% — low; expected during off-peak hours
```

**Workers per pod:** 4 uvicorn workers on 8 vCPUs. ONNX Runtime thread pool is
set to 2 threads per session (8 threads total per pod = 8 vCPU count). This
avoids thread contention while keeping all cores used.

## Autoscale policy

| Parameter | Value |
|---|---|
| Trigger | CPU utilisation > 60% sustained for 90 s |
| Scale-up increment | +2 replicas |
| Scale-down increment | -1 replica |
| Scale-down cooldown | 5 minutes |
| Minimum replicas | 6 |
| Maximum replicas | 12 |

Autoscale will not trigger during normal peak load (800 RPS, ~45% CPU). It
activates when traffic exceeds ~1,070 RPS (the RPS that drives 6-replica
utilisation to 60%). At 12 replicas, the ceiling before CPU saturation is
~2,130 RPS — well beyond any realistic spike for this service.

## Redis sizing

**Feature store (user features):**
- ElastiCache r6g.large: 13.07 GB memory per node, 2-node cluster
- Feature vector per user: 512 floats × 4 bytes = 2 KB
- 1 million active users: 1M × 2 KB = 2 GB in Redis
- 2 GB / 13 GB = 15% utilisation — ample headroom for growth to 6.5M users
- Replication: primary + replica for automatic failover (<60 s)

**ANN item index:**
- 500,000 items × 256 floats × 4 bytes = ~512 MB
- Fits comfortably in the same cluster on a separate Redis DB index

## Pod startup and rolling deploy

The model artifact is mounted from S3 at pod init (ADR 0002). This is not
baked into the container image. Startup sequence:

| Phase | Duration |
|---|---|
| S3 download (180 MB ONNX, VPC endpoint) | ~12 s |
| ONNX Runtime model load (INT8 CPU) | ~5 s |
| Uvicorn startup + readiness probe delay | ~3 s |
| **Total pod cold-start** | **~20 s** |
| Readiness probe start-period (Dockerfile) | 40 s (with margin) |

Rolling deploy configuration:
- `maxUnavailable: 1`, `maxSurge: 1`
- 6 replicas → each pod rolled serially with one in-progress
- Full rollout time: 6 pods × 40 s = approximately **4 minutes**
- At no point does available capacity drop below 5/6 replicas (83%)

## Failure modes

| Failure | Impact | Recovery |
|---|---|---|
| 1 pod OOMKilled | 5/6 replicas, capacity drops to ~833 RPS | EKS replaces pod; autoscale adds if needed |
| 1 AZ unavailable (2 pods) | 4/6 replicas, ~667 RPS capacity | Covers sustained load (350 RPS); autoscale compensates for peak |
| Redis failover | <60 s of cold-start degradation (warm users treated as cold-start) | Automatic ElastiCache failover |
| S3 unreachable at pod init | New pods fail readiness probe; existing pods continue serving | S3 HA within region; incident if S3 region-level outage |
| Model version mismatch | Pod serving wrong version; monitored by `ModelVersionMismatch` alert | Rollback runbook; see runbooks/rollback.md |

## Monthly cost

| Component | Configuration | Monthly cost |
|---|---|---|
| Compute (baseline, 6 pods) | 6 × c6i.2xlarge × $0.340/hr × 730 hr | $1,490 |
| Redis feature + ANN store | ElastiCache r6g.large × 2 | $180 |
| S3 model artifacts + API GW | Storage + requests | $120 |
| CloudWatch metrics + logs | Standard ingestion + storage | $80 |
| ALB + data transfer | ~350 GB/month egress estimate | $100 |
| ECR image storage | < 5 GB | $5 |
| **Baseline total** | | **~$1,975 → ~$2,200 with buffer** |
| **Full scale (12 pods)** | Additional 6 pods compute | **~$3,500 → ~$3,800 with buffer** |