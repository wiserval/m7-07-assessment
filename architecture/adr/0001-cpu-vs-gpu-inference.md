# ADR 0001 — CPU inference over GPU for the Recommender Service

**Date:** 2026-06-05
**Status:** Accepted
**Deciders:** ML Platform lead, Infrastructure lead

## Context

The Recommender Service must serve inference at up to 800 RPS with a p95
end-to-end latency SLO of 100 ms. The inference workload is a two-stage
pipeline: ANN retrieval (~5 ms, CPU-bound) followed by an ONNX MLP ranker
scoring 200 candidates per request (~20–25 ms on CPU). Requests are served
synchronously — one HTTP response per request, no cross-request batching.

The question is whether to run the ranker on CPU instances (c6i.2xlarge)
or GPU instances (g4dn.xlarge, NVIDIA T4).

## Decision

Run inference on CPU. Deploy six c6i.2xlarge replicas at baseline, autoscale
to twelve.

## Rationale

**Batch size.** GPU inference is efficient when the batch size is large enough
to saturate CUDA parallelism and amortize PCIe data transfer overhead. At
800 RPS with a 5 ms batching window, the effective batch size at the ranker
is ~4 requests (800 × 0.005). A T4 reaches GPU efficiency at batch sizes
of 32+. At batch-4, CPU ONNX inference is faster than GPU inference for
this model shape because the PCIe round-trip dominates.

**Throughput arithmetic (CPU path).**
- Inference time per request (ONNX, INT8 quantized): 22 ms
- CPU-seconds needed at 1,000 RPS (headroom target): 1,000 × 0.022 = 22 cores
- At 60% target utilization: 22 / 0.60 = 36.7 vCPUs required
- c6i.2xlarge = 8 vCPU → ceiling(36.7 / 8) = 5 replicas → 6 with safety margin

**Cost comparison (6 instances, 730 hr/month).**

| Option     | Instance      | $/hr   | Monthly  |
|------------|---------------|--------|----------|
| CPU        | c6i.2xlarge   | $0.340 | $1,490   |
| GPU        | g4dn.xlarge   | $0.526 | $2,304   |

GPU costs $814/month more for worse latency at this batch size.

**Latency budget check (CPU path).**

| Segment                        | Budget |
|-------------------------------|--------|
| ALB + TLS                     | 5 ms   |
| Feature lookup (Redis)        | 8 ms   |
| ANN retrieval                 | 5 ms   |
| ONNX MLP ranker               | 22 ms  |
| Serialization + egress        | 10 ms  |
| Total                         | 50 ms  |
| Remaining margin to 100 ms SLO| 50 ms  |

CPU path hits ~50 ms P50, leaving 50 ms of margin before the internal SLO
and 70 ms of margin before the scenario's 120 ms SLA.

## Alternatives considered

**g4dn.xlarge (NVIDIA T4).** Provides GPU headroom for future model complexity
increases. Rejected because the current model does not benefit from GPU at
this batch size, and the latency SLO is met comfortably on CPU.

**inf2.xlarge (AWS Inferentia2).** Potentially cheaper than T4 at sustained
throughput with good batching characteristics. Requires model compilation to
Neuron format, adding toolchain complexity and breaking local ONNX development
workflows. Rejected given CPU already meets requirements.

## Consequences

- Pod startup is faster without CUDA initialization.
- If the ranker is replaced by a model where CPU ONNX inference exceeds 30 ms
  per request, this decision must be revisited. The batch-size argument holds
  only while per-request inference fits within the latency budget on CPU.
- At 12 replicas (autoscale ceiling), the system handles ~2,000 RPS before
  CPU saturation. Traffic beyond that level should trigger architectural review
  — at that scale, a batching layer (e.g., Triton Inference Server with
  dynamic batching) may justify the GPU investment.