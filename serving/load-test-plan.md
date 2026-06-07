# Load Test Plan — Recommender Service (Scenario X)

## Tool

**k6** (Grafana k6). Chosen for scripted load generation, built-in histogram
metrics, native Prometheus remote-write output, and consistency with the
project's existing test infrastructure.

## Target environment

All tests run against the **staging** environment
(`https://recommendations.internal.staging`). Never run spike or soak tests
against production. The staging Redis feature store is pre-populated with
a representative 100k-user corpus (10% of the production user base) generated
from anonymised production feature snapshots.

## Pass/fail criteria

Tests fail if any of the following conditions are observed. These thresholds
map directly to the SLOs defined in `serving/slos.yaml`.

| Metric | Threshold | Corresponding SLO |
|---|---|---|
| p95 latency (sync endpoint) | > 100 ms | P95Latency SLO |
| p99 latency (sync endpoint) | > 150 ms | P99Latency SLO |
| HTTP 5xx error rate | > 0.5% | Availability SLO |
| HTTP 5xx error rate (spike peak) | > 1.44% | AvailabilityFastBurn threshold |
| Successful request rate | < 99.5% | Availability SLO |

## Test scenarios

### Scenario 1 — Baseline (smoke + validation)

**Purpose:** Confirm the service is healthy and the SLO holds under sustained
load before running heavier tests.

| Parameter | Value |
|---|---|
| Load shape | Constant |
| RPS | 350 (sustained average) |
| Duration | 10 minutes |
| VUs (Little's Law: 350 × 0.050 s) | 18 |
| Warm user ratio | 80% |
| Cold-start user ratio | 20% |

**Pass criteria:** p95 < 80 ms, p99 < 120 ms, error rate < 0.1%.
If this test fails, do not proceed to peak or spike tests.

---

### Scenario 2 — Peak load

**Purpose:** Confirm the SLO holds at the scenario-defined peak of 800 RPS.

| Parameter | Value |
|---|---|
| Load shape | Ramp-up then constant |
| Ramp-up | 0 → 800 RPS over 2 minutes |
| Sustain | 800 RPS for 10 minutes |
| Ramp-down | 800 → 0 RPS over 1 minute |
| VUs at peak | 40 |

**Pass criteria:** p95 < 100 ms, p99 < 150 ms, error rate < 0.5%.
Observe autoscale behaviour: CPU should reach ~45% at 800 RPS on 6 replicas.
Autoscale must not trigger (trigger threshold is ~1,070 RPS).

---

### Scenario 3 — Spike (above sizing target)

**Purpose:** Validate that autoscale activates correctly when traffic exceeds
the sizing target, and that the SLO recovers after the spike.

| Parameter | Value |
|---|---|
| Load shape | Step spike |
| Pre-spike | 350 RPS for 3 minutes |
| Spike | Jump to 1,200 RPS instantly |
| Spike duration | 5 minutes |
| Post-spike | 350 RPS for 5 minutes |
| VUs at spike | 60 |

**Observed behaviour expected:**
1. At spike onset: p95 latency rises; error rate may briefly exceed 0.5%.
2. Within 90 s: HPA detects CPU > 60%, adds 2 replicas.
3. By minute 3 of spike: p95 returns below 100 ms as new pods pass readiness probe.

**Pass criteria:**
- p95 exceeds 100 ms for no more than 3 consecutive minutes.
- Error rate never exceeds the `AvailabilityFastBurn` threshold of 1.44%.
- p95 returns below 100 ms within 3 minutes of autoscale pod becoming ready.
- Post-spike: p95 < 80 ms within 2 minutes of load returning to 350 RPS.

---

### Scenario 4 — Soak (degradation detection)

**Purpose:** Detect memory leaks, connection pool exhaustion, or Redis
connection churn over an extended run.

| Parameter | Value |
|---|---|
| Load | 350 RPS constant |
| Duration | 2 hours |
| VUs | 18 |

**Pass criteria:**
- p95 latency must not increase by more than 15 ms over the 2-hour window.
- Memory per pod must not grow by more than 200 MB over the run.
- Error rate must remain < 0.1% throughout.
- No pod restart events during the test.

---

### Scenario 5 — Cold-start ratio stress

**Purpose:** Confirm that a high cold-start ratio (new user influx during
a marketing campaign) does not degrade warm-user latency.

| Parameter | Value |
|---|---|
| Load | 800 RPS |
| Duration | 5 minutes |
| Cold-start user ratio | 60% (vs baseline 20%) |
| VUs | 40 |

Cold-start requests skip Redis and the ranker entirely, but they do hit
Redis for the popularity list. This tests whether the popularity cache
path degrades the feature store's response time for warm users.

**Pass criteria:** p95 latency for warm-user requests < 100 ms throughout.
Cold-start request p95 < 20 ms (popularity list lookup only).

## k6 implementation notes

- Use `k6/http` with `scenarios` to generate the exact load shapes above.
- Tag requests by user type: `tags: { user_type: "warm" }` / `"cold_start"`.
  This allows filtering latency results by path in Grafana.
- Pre-generate a 100k warm user ID corpus and a 10k cold-start user ID corpus
  before running tests. Load from CSV using k6's `SharedArray`.
- Export metrics to Prometheus via `K6_PROMETHEUS_RW_SERVER_URL` and view
  alongside production dashboards in Grafana to confirm thresholds.
- Run with `--out experimental-prometheus-rw` for native push to the
  staging Prometheus instance.

## Endpoint coverage

Only the sync endpoint (`GET /v1/recommendations/{user_id}`) is load tested.
The batch and async endpoints are tested for correctness in integration tests,
not load tested — they are not on the home-screen load path and do not
contribute to the p95 SLO defined in `serving/slos.yaml`.