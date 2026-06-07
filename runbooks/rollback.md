# Rollback Runbook — Recommender Service

## Trigger conditions

Execute this runbook when any of these alerts fire in Grafana / PagerDuty.
Thresholds here must match `monitoring/alerts.yaml` exactly.

| Alert | Threshold | Severity |
|---|---|---|
| `AvailabilityFastBurn` | 5xx error rate > 1.44% in both 1h + 5m windows | Page |
| `LatencySLABreach` | p95 > 120 ms for ≥ 5 min | Page |
| `ModelVersionMismatch` | > 1 distinct `X-Model-Version` across pods for ≥ 10 min | Page |
| `PredictionDistributionDrift` | KL divergence > 0.15 for ≥ 30 min | Ticket — investigate first |

## Prerequisites

```bash
# Run once per session before executing any step below
aws eks update-kubeconfig --name $EKS_CLUSTER_NAME --region us-east-1
export MLFLOW_TRACKING_URI="https://mlflow.internal.prod/recommender"
```

## Steps

**1. Acknowledge the PagerDuty alert.** Do not resolve yet.

**2. Confirm the alert is real.**
Open Grafana → Recommender dashboard → check `p95 latency` and `5xx rate` panels.
If both look normal (spike already cleared), monitor for 5 minutes before continuing.

**3. Check what version is currently serving.**
```bash
curl -sI \
  "https://recommendations.internal.prod/v1/recommendations/smoke-test-user?limit=1" \
  | grep -i x-model-version
```
Note the value: `ranker/v{semver}+{run-id}`. This is the current version.

**4. Find the previous Production version in MLflow.**
Go to `https://mlflow.internal.prod/recommender` → Models → ranker → Versions.
Identify the most recent version with `current_stage = Production` that is **not**
the version from step 3. Note its full version string.

**5. Execute the rollback.**
```bash
PREV="ranker/v{previous_semver}+{previous_run_id}"   # from step 4

kubectl set env deployment/recommender-service \
  MODEL_S3_URI="s3://prod-models/${PREV}/model.onnx" \
  MODEL_VERSION="${PREV}" \
  -n production

kubectl rollout status deployment/recommender-service \
  -n production --timeout=300s
```

**6. Verify the rollback succeeded.**
```bash
curl -sI \
  "https://recommendations.internal.prod/v1/recommendations/smoke-test-user?limit=1" \
  | grep -i x-model-version
```
Output must show `${PREV}`. If it still shows the old version, pods are still
rolling — wait 60 s and retry.

**7. Confirm alerts have cleared.**
Wait 5 minutes. Check Grafana — `AvailabilityFastBurn` and `LatencySLABreach`
must show as resolved before proceeding.

**8. Notify the team.**
Post to `#ml-platform`:
> Rollback executed at [time]. Now serving `${PREV}`.
> Previous version: `{version from step 3}`. Incident: [PagerDuty link].

**9. Resolve the PagerDuty incident.** Only after step 7 confirms alerts cleared.

**10. File an incident review.**
Open a GitHub issue: `Incident [date] — rollback from {version}`.
Tag `ml-platform`. Link this runbook and the PagerDuty timeline.

---

## If rollback does not resolve the alert within 15 minutes

```bash
# Check Redis health
redis-cli -h $REDIS_HOST -p 6379 ping

# Check pod logs for inference errors
kubectl logs -l app=recommender -n production --tail=100 | grep -i error

# Check pod resource pressure
kubectl top pods -n production -l app=recommender
```

If unresolved after 30 minutes, page the ML Platform Lead directly.