---
title: 5xxErrorRate
---

# Alert: 5xx-error-rate

## Alert Quick Info

| **Alert name** | cf-*-5xx-error-rate (CloudWatch alarm) |
| --- | --- |
| Service type | CloudFront distribution — origin 5xx errors |
| Impact Scope | User-facing — users receiving 5xx errors on the affected domain |
| Transient possibility | Yes (brief origin saturation during load spikes) |

## What is this ?

This is a CloudWatch alarm that fires when a CloudFront distribution's 5xx error rate exceeds 3% for 2 consecutive minutes. Unlike the Prometheus-based `CloudFront5xxErrorRateAnomalyDetection` alert, this alarm provides a top-10 breakdown of the failing URIs with status codes and request counts — use this to identify which backend service is failing.

## Noise triage ?

**Not noise by default.** 5xx means the origin is failing to serve user requests. However:

1. **Short spike (< 5 min) with self-recovery**: often caused by a brief backend saturation during load spike, followed by HPA scale-up. If the alarm transitions back to OK quickly, this is transient — still document the event.
2. **Low absolute count**: if the total 5xx count is in single digits on a low-traffic distribution, the percentage can spike without real user impact.
3. **Single URI pattern**: if only one URI path is affected, the blast radius may be limited to one feature rather than the whole site.

Quick Tip:

- `Transient`: alarm resolved within 5 minutes, no user reports, HPA scaled up
- `Investigate`: sustained alarm (> 5 min), multiple URI patterns, or high absolute counts
- `Escalate`: main domain affected (www.sportybet.com), payment/bet paths in the top URIs, or user reports confirmed

## Action if not noise

1. **Read the top URIs** from the alert body — they tell you which backend service is failing, example:
   - `/api/ng/factsCenter/*` → facts-center-query service (Nigeria bet platform)
   - `/api/ng/realSportsGame/*` → real sports game service
   - `/api/media/v*/*` → media platform service
   - `/api/ng/payment/*` or `/api/ng/pocket/*` → payment services (highest priority)

2. **Identify the CC from the URI path** (e.g. `/api/ng/` = Nigeria, `/api/gh/` = Ghana) and resolve the cluster.
3. **Check the backend pods** for the affected identified service.
4. **Check pod logs** for saturation indicators:
   Tip: Look for busy thread pool exhaustion, connection pool timeouts, OOM events, database connection errors.
5. **Check HPA status** — is the service scaling up in response to load?

6. **Check CloudFront metrics** to see if the 5xx rate is dropping:
   ```bash
   aws cloudwatch get-metric-statistics \
     --namespace AWS/CloudFront --metric-name 5xxErrorRate \
     --dimensions Name=DistributionId,Value=<DIST_ID> Name=Region,Value=Global \
     --start-time $(date -u -v-1H +%Y-%m-%dT%H:%M:%S) \
     --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
     --period 60 --statistics Average \
     --profile sporty-sso-ro --region us-east-1
   ```

7. **If sustained**: escalate to the service team owning the top-offending URIs. Include the alarm name, time window, top URIs, and pod/HPA state in the escalation.

8. **If self-resolved**: document the timeline, peak error rate, affected URIs, and whether HPA scale-up correlated with recovery.

### External links

- [Service-team escalation sheet](https://docs.google.com/spreadsheets/d/18f6Mfkv5lo7JX4_J7lqQXnjHO7UsfT5Ro0l16L6WKV4/edit?gid=1372267029#gid=1372267029)
- [Triage and Alert Index — Confluence](https://opennetltd.atlassian.net/wiki/spaces/DT/pages/4423778381/Triage+And+Alert+Index)

- [All runbooks](/runbooks/)
- [Home](/)