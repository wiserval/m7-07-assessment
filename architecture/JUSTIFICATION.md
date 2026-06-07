# Architecture Justification

## Pattern: synchronous REST serving with two-stage retrieval and ranking

This is a synchronous request-response service because the product requirement
leaves no other option: personalized recommendations must appear before the
home screen renders. The user is waiting. An async pre-computation pattern —
computing recommendations in a background worker and caching them per user —
was considered and rejected. It would mean serving recommendations computed
from yesterday's browse history to a user whose last ten minutes of activity
signal something completely different. The 30-day personalization window only
works if feature retrieval happens at request time.

## Two-stage over single-stage inference

A single-stage approach — scoring all ~500k catalog items per request — fails
the latency budget immediately. At 800 RPS, even a 1 ms per-item score would
require 500 seconds of compute per second of traffic. The two-stage pattern
resolves this: an approximate nearest-neighbor index narrows the field to 200
candidates in ~5 ms, then the ONNX MLP ranker scores those 200 in ~22 ms. The
precision loss from ANN approximation is acceptable and is standard practice
at this catalog scale.

## CPU over GPU

Derived from batch size arithmetic. At 800 RPS with synchronous per-request
serving, the effective batch size reaching the ranker at any instant is
approximately 4 (800 RPS × 0.005 s batching window). A T4 GPU requires
batches of 32+ to amortize PCIe transfer overhead and show throughput gains
over CPU. Below that threshold, CPU ONNX inference is faster and a c6i.2xlarge
is 55% cheaper than a g4dn.xlarge. Six c6i.2xlarge replicas provide 48 vCPUs,
handling 1,000 RPS at ~48% average utilization with headroom to autoscale to
twelve before any SLO is at risk. Full decision record in
[ADR 0001](adr/0001-cpu-vs-gpu-inference.md).

## Mount over bake for model artifacts

The scenario requires continuous A/B testing. Baking a 180 MB model into the
container image ties model release cadence to container build cadence — every
experiment variant needs a separate image build, push, and rolling deploy,
adding ~15 minutes of friction per variant. Mounting from S3 at pod startup
decouples the two release cycles entirely. A new experiment variant is live in
~2 minutes (update the MODEL_S3_URI environment variable, roll the pods). The
container image tracks code changes; the model registry tracks ML training
runs. These are not the same thing and should not be the same artifact.
Full decision record in [ADR 0002](adr/0002-bake-vs-mount-model.md).

## Redis over DynamoDB for the feature store

Both were evaluated. DynamoDB has lower operational overhead and better
durability guarantees. Redis (ElastiCache r6g.large × 2) has sub-millisecond
P99 read latency on small KV items at our access pattern. DynamoDB P99 for
the same access pattern with a hot-key distribution is 5–15 ms.

The end-to-end budget is 120 ms with a 20 ms internal margin. Spending up to
15 ms on a single feature lookup is too expensive. Redis wins on latency.
The durability trade-off is acceptable: the feature store is rebuilt hourly
by the feature pipeline. A Redis failure causes cold-start degradation for
at most one hour, not data loss.

## Stateless horizontal scaling

All persistent state lives in Redis and S3. The Recommender Service pods carry
no local state between requests, which makes horizontal autoscaling
straightforward. The EKS HPA scales on CPU utilization > 60% over a 90-second
window. No session affinity, no sticky routing required.

## Rejected alternatives

**Lambda / serverless inference.** Cold start for a Python process loading a
180 MB ONNX model is 200–800 ms without provisioned concurrency, which is
incompatible with the 100 ms SLO. Provisioned concurrency at 350 sustained
RPS costs more than the EKS approach with less operational visibility.

**Pre-computed recommendation cache.** Eliminates inference latency, but
serves stale recommendations disconnected from real-time user context. Used
only for cold-start users where there are no real-time signals to work with.

**Stream-based serving via Kafka consumers.** Adds ordering guarantees and
delivery semantics that provide no benefit for a synchronous read path.
Rejected without hesitation.