# Chapter 9: PromQL Advanced — 100+ Production Queries

> **Part 5 — PromQL Masterclass**

---

## 1. Objective

By the end of this chapter you will have a categorized, ready-to-use library of 100+ real production PromQL queries — the same kind of query library a platform team keeps internally — covering cluster health, node resources, container resources, Kubernetes object state, application RED metrics, SLO/error-budget math, and query performance optimization. You will also understand the handful of optimization techniques that keep these queries fast as your cluster grows.

Every query below assumes the metric names produced by your Chapter 5 kube-prometheus-stack install (Node Exporter, cAdvisor via kubelet, kube-state-metrics) — you'll be able to run nearly all of them against your own cluster right now. Application-level queries (RED for Online Boutique) will return real data once you complete Part 10; they're included here so you have the reference ready when you get there.

---

## 2. Concept — The Query Library

### 2.1 Cluster Health & Availability

```promql
# 1. Are all Prometheus scrape targets currently healthy?
up == 0

# 2. Count of targets down, grouped by job
count by (job) (up == 0)

# 3. Percentage of targets currently up, per job
sum by (job) (up) / count by (job) (up) * 100

# 4. Nodes currently NotReady
kube_node_status_condition{condition="Ready", status="true"} == 0

# 5. Total node count vs Ready node count
count(kube_node_info)
count(kube_node_status_condition{condition="Ready", status="true"} == 1)

# 6. Pods not in Running phase, by namespace
count by (namespace) (kube_pod_status_phase{phase!="Running"} == 1)

# 7. Cluster-wide pod restart rate (last hour)
sum(increase(kube_pod_container_status_restarts_total[1h]))

# 8. Prometheus's own health: TSDB head series (cardinality watch)
prometheus_tsdb_head_series

# 9. Prometheus's own scrape duration (is Prometheus itself falling behind?)
histogram_quantile(0.99, sum by (le) (rate(prometheus_target_interval_length_seconds_bucket[5m])))

# 10. Alertmanager: currently firing alert count
sum(ALERTS{alertstate="firing"})
```

### 2.2 Node Resources — USE Method (Node Exporter)

```promql
# 11. Node CPU utilisation (%) — USE: Utilization
100 * (1 - avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])))

# 12. Per-core CPU utilisation breakdown by mode
sum by (instance, mode) (rate(node_cpu_seconds_total[5m]))

# 13. Node CPU saturation: load average vs core count — USE: Saturation
node_load1 / count by (instance) (node_cpu_seconds_total{mode="idle"})

# 14. Node memory utilisation (%)
100 * (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes))

# 15. Node memory saturation: is the kernel under memory pressure?
rate(node_vmstat_pgmajfault[5m])   # major page faults = disk-backed paging, a saturation red flag

# 16. Node filesystem usage (%) per mountpoint
100 * (1 - (node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"} / node_filesystem_size_bytes{fstype!~"tmpfs|overlay"}))

# 17. Node filesystem: time-to-full prediction (hours), a leading indicator
(node_filesystem_avail_bytes / deriv(node_filesystem_avail_bytes[1h])) / 3600 < 0

# 18. Node disk I/O saturation: average queue size
rate(node_disk_io_time_weighted_seconds_total[5m])

# 19. Node disk read/write throughput (bytes/sec)
rate(node_disk_read_bytes_total[5m])
rate(node_disk_written_bytes_total[5m])

# 20. Node network receive/transmit throughput (bytes/sec)
rate(node_network_receive_bytes_total{device!="lo"}[5m])
rate(node_network_transmit_bytes_total{device!="lo"}[5m])

# 21. Node network errors — USE: Errors
rate(node_network_receive_errs_total[5m]) + rate(node_network_transmit_errs_total[5m])

# 22. Node network drops (buffer saturation)
rate(node_network_receive_drop_total[5m]) + rate(node_network_transmit_drop_total[5m])

# 23. Node TCP connection states (established, time_wait, etc.)
node_netstat_Tcp_CurrEstab

# 24. Node file descriptor usage (%)
node_filefd_allocated / node_filefd_maximum * 100

# 25. Node clock skew / time sync drift (a surprisingly common silent incident cause)
node_timex_offset_seconds
```

### 2.3 Container Resources — USE Method (cAdvisor, Part 6 deep dive)

```promql
# 26. Container CPU usage (cores) per pod
sum by (pod) (rate(container_cpu_usage_seconds_total{container!=""}[5m]))

# 27. Container CPU usage as % of its own CPU limit
sum by (pod) (rate(container_cpu_usage_seconds_total{container!=""}[5m]))
/ sum by (pod) (kube_pod_container_resource_limits{resource="cpu"})

# 28. Container CPU throttling — USE: Saturation (the metric most beginners miss)
sum by (pod) (rate(container_cpu_cfs_throttled_periods_total[5m]))
/ sum by (pod) (rate(container_cpu_cfs_periods_total[5m]))

# 29. Container memory working set (the number the OOM killer actually watches)
sum by (pod) (container_memory_working_set_bytes{container!=""})

# 30. Container memory usage as % of its own memory limit
sum by (pod) (container_memory_working_set_bytes{container!=""})
/ sum by (pod) (kube_pod_container_resource_limits{resource="memory"})

# 31. Containers OOMKilled in the last hour
increase(kube_pod_container_status_last_terminated_reason{reason="OOMKilled"}[1h]) > 0

# 32. Container restart count, by pod
sum by (pod) (kube_pod_container_status_restarts_total)

# 33. Container filesystem usage (ephemeral storage)
container_fs_usage_bytes{container!=""}

# 34. Container network receive/transmit rate, by pod
sum by (pod) (rate(container_network_receive_bytes_total[5m]))
sum by (pod) (rate(container_network_transmit_bytes_total[5m]))

# 35. Top 10 pods by CPU usage cluster-wide
topk(10, sum by (pod, namespace) (rate(container_cpu_usage_seconds_total{container!=""}[5m])))

# 36. Top 10 pods by memory usage cluster-wide
topk(10, sum by (pod, namespace) (container_memory_working_set_bytes{container!=""}))

# 37. Pods with CPU requests but no limits set (a governance/best-practice check)
kube_pod_container_resource_requests{resource="cpu"} unless kube_pod_container_resource_limits{resource="cpu"}
```

### 2.4 Kubernetes Object State (kube-state-metrics, Part 8 deep dive)

```promql
# 38. Deployments where available replicas < desired replicas
kube_deployment_status_replicas_available < kube_deployment_spec_replicas

# 39. StatefulSets not fully rolled out
kube_statefulset_status_replicas_ready != kube_statefulset_status_replicas

# 40. DaemonSet pods not scheduled on all eligible nodes
kube_daemonset_status_desired_number_scheduled - kube_daemonset_status_current_number_scheduled > 0

# 41. Pods stuck in Pending, by namespace
count by (namespace) (kube_pod_status_phase{phase="Pending"} == 1)

# 42. Pods in CrashLoopBackOff (waiting reason)
kube_pod_container_status_waiting_reason{reason="CrashLoopBackOff"} == 1

# 43. PVCs not Bound
kube_persistentvolumeclaim_status_phase{phase!="Bound"} == 1

# 44. PersistentVolume usage close to full (needs kubelet volume stats, cross-referenced)
kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes > 0.85

# 45. HPA currently at max replicas (can't scale further — a capacity warning)
kube_horizontalpodautoscaler_status_current_replicas
  == kube_horizontalpodautoscaler_spec_max_replicas

# 46. Jobs that failed
kube_job_status_failed > 0

# 47. CronJobs that haven't run recently (missed schedule)
time() - kube_cronjob_status_last_schedule_time > 3600  # no run in the last hour

# 48. Nodes with disk pressure condition
kube_node_status_condition{condition="DiskPressure", status="true"} == 1

# 49. Nodes with memory pressure condition
kube_node_status_condition{condition="MemoryPressure", status="true"} == 1

# 50. Secrets/ConfigMaps count per namespace (governance/audit visibility)
count by (namespace) (kube_secret_info)
count by (namespace) (kube_configmap_info)

# 51. Pods with no resource requests at all (governance check)
count(kube_pod_container_info) - count(kube_pod_container_resource_requests{resource="cpu"})
```

### 2.5 Application RED Metrics (Part 10, once Online Boutique is instrumented)

```promql
# 52. Request rate, per service
sum by (job) (rate(http_requests_total[5m]))

# 53. Error rate, per service (%)
sum by (job) (rate(http_requests_total{status=~"5.."}[5m]))
/ sum by (job) (rate(http_requests_total[5m])) * 100

# 54. p50 / p90 / p99 latency, per service
histogram_quantile(0.50, sum by (le, job) (rate(http_request_duration_seconds_bucket[5m])))
histogram_quantile(0.90, sum by (le, job) (rate(http_request_duration_seconds_bucket[5m])))
histogram_quantile(0.99, sum by (le, job) (rate(http_request_duration_seconds_bucket[5m])))

# 55. Requests per second, by HTTP status class
sum by (status) (rate(http_requests_total[5m]))

# 56. gRPC error rate (Online Boutique is mostly gRPC internally)
sum by (grpc_service) (rate(grpc_server_handled_total{grpc_code!="OK"}[5m]))
/ sum by (grpc_service) (rate(grpc_server_handled_total[5m]))

# 57. In-flight/concurrent requests (a saturation-adjacent RED extension)
sum by (job) (http_requests_in_flight)

# 58. Slowest 1% of requests specifically (tail-focused, not just p99)
histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket{job="checkoutservice"}[5m])))
  - histogram_quantile(0.50, sum by (le) (rate(http_request_duration_seconds_bucket{job="checkoutservice"}[5m])))
```

### 2.6 SLO / Error Budget Queries (direct implementation of Chapter 3)

```promql
# 59. Availability SLI (ratio of good requests), rolling 5m
sum(rate(http_requests_total{job="checkoutservice", status!~"5.."}[5m]))
/ sum(rate(http_requests_total{job="checkoutservice"}[5m]))

# 60. Availability SLI, rolling 28d (for SLO compliance reporting)
sum(rate(http_requests_total{job="checkoutservice", status!~"5.."}[28d]))
/ sum(rate(http_requests_total{job="checkoutservice"}[28d]))

# 61. Error budget remaining (%), given a 99.9% SLO over 28d
(1 - (
  sum(increase(http_requests_total{job="checkoutservice", status=~"5.."}[28d]))
  / sum(increase(http_requests_total{job="checkoutservice"}[28d]))
) / (1 - 0.999)) * 100

# 62. Fast burn rate (1h window) — for immediate paging (Part 12)
(
  sum(rate(http_requests_total{job="checkoutservice", status=~"5.."}[1h]))
  / sum(rate(http_requests_total{job="checkoutservice"}[1h]))
) / (1 - 0.999)

# 63. Slow burn rate (6h window) — for ticket-level, non-paging alerts
(
  sum(rate(http_requests_total{job="checkoutservice", status=~"5.."}[6h]))
  / sum(rate(http_requests_total{job="checkoutservice"}[6h]))
) / (1 - 0.999)

# 64. Latency SLI: % of requests under 300ms threshold
sum(rate(http_request_duration_seconds_bucket{job="checkoutservice", le="0.3"}[5m]))
/ sum(rate(http_request_duration_seconds_count{job="checkoutservice"}[5m]))
```

### 2.7 Capacity Planning & Trend Queries

```promql
# 65. Predicted node disk-full time (linear extrapolation, 4h ahead)
predict_linear(node_filesystem_avail_bytes{mountpoint="/"}[6h], 4 * 3600) < 0

# 66. Predicted memory exhaustion (Gauge trend prediction)
predict_linear(node_memory_MemAvailable_bytes[6h], 4 * 3600) < 0

# 67. Week-over-week request volume growth (%)
(sum(rate(http_requests_total[1h])) - sum(rate(http_requests_total[1h] offset 7d)))
/ sum(rate(http_requests_total[1h] offset 7d)) * 100

# 68. Cluster-wide CPU request vs actual usage (rightsizing signal, ties to Goldilocks/VPA, Part 18)
sum(kube_pod_container_resource_requests{resource="cpu"})
- sum(rate(container_cpu_usage_seconds_total{container!=""}[5m]))

# 69. Namespace resource quota utilisation (%)
kube_resourcequota{type="used", resource="requests.cpu"}
/ kube_resourcequota{type="hard", resource="requests.cpu"} * 100

# 70. Node count trend (cluster autoscaling visibility)
count(kube_node_info)
```

### 2.8 Kubernetes Control Plane Health

```promql
# 71. API server request latency p99
histogram_quantile(0.99, sum by (le, verb) (rate(apiserver_request_duration_seconds_bucket[5m])))

# 72. API server error rate (5xx)
sum(rate(apiserver_request_total{code=~"5.."}[5m])) / sum(rate(apiserver_request_total[5m]))

# 73. etcd request latency p99 (control plane health, relevant on self-managed clusters)
histogram_quantile(0.99, sum by (le) (rate(etcd_request_duration_seconds_bucket[5m])))

# 74. Scheduler: pods failing to schedule
increase(scheduler_pod_scheduling_attempts_total{result="error"}[5m])

# 75. CoreDNS query error rate (a very common, often-overlooked failure domain)
sum(rate(coredns_dns_responses_total{rcode!="NOERROR"}[5m]))
/ sum(rate(coredns_dns_responses_total[5m]))

# 76. CoreDNS latency p99
histogram_quantile(0.99, sum by (le) (rate(coredns_dns_request_duration_seconds_bucket[5m])))

# 77. Webhook (admission controller) latency — a common silent source of pod creation delay
histogram_quantile(0.99, sum by (le) (rate(apiserver_admission_webhook_admission_duration_seconds_bucket[5m])))
```

### 2.9 Alerting-Support Queries

```promql
# 78. Currently firing alerts, by severity
count by (severity) (ALERTS{alertstate="firing"})

# 79. Alerts that have been firing longest (possible stuck/ignored alerts)
(time() - ALERTS_FOR_STATE) and ALERTS{alertstate="firing"}

# 80. Silenced alert count (Alertmanager metric, Part 12)
alertmanager_silences{state="active"}

# 81. Notification delivery failures (Alertmanager metric, Part 12)
rate(alertmanager_notifications_failed_total[5m])
```

### 2.10 Storage & Volumes

```promql
# 82. PVC usage (%) per claim
kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes * 100

# 83. PVC inode usage (%) — a sneaky failure mode disk-% alone misses
kubelet_volume_stats_inodes_used / kubelet_volume_stats_inodes * 100

# 84. StorageClass usage summary, by class
sum by (storageclass) (kube_persistentvolume_capacity_bytes)

# 85. Orphaned PVs (Released but not reclaimed)
kube_persistentvolume_status_phase{phase="Released"} == 1
```

### 2.11 Real-world "Debugging" One-Liners

```promql
# 86. Every distinct label value for a given label on a metric (great for exploring cardinality)
count by (job) (up)

# 87. Which targets a specific job actually resolved to, and their health
up{job="node-exporter"}

# 88. Scrape duration per target (finding a slow-to-scrape exporter)
scrape_duration_seconds

# 89. Number of samples a single scrape produced (a per-target cardinality check)
scrape_samples_scraped

# 90. Find metrics whose name matches a pattern (useful when you don't remember exact names)
{__name__=~"container_memory.*"}

# 91. Absence detection — did a metric disappear entirely? (different from value=0)
absent(up{job="checkoutservice"})

# 92. Rate of change comparison, this hour vs last hour
rate(http_requests_total[5m]) - rate(http_requests_total[5m] offset 1h)
```

### 2.12 Node Exporter Deep Cuts (previewed here, fully covered in Part 7)

```promql
# 93. Systemd units that failed
node_systemd_unit_state{state="failed"} == 1

# 94. NUMA memory imbalance signal
node_memory_numa_MemFree

# 95. Hugepages usage
node_memory_HugePages_Total - node_memory_HugePages_Free

# 96. Interrupt rate (a CPU saturation-adjacent signal on network-heavy nodes)
rate(node_intr_total[5m])

# 97. Entropy availability (can starve crypto/TLS-heavy workloads on some kernels)
node_entropy_available_bits
```

### 2.13 Cost & Efficiency (previewed here, expanded with OpenCost in Part 18)

```promql
# 98. Requested vs. actually used CPU, cluster-wide (waste signal)
(sum(kube_pod_container_resource_requests{resource="cpu"}) - sum(rate(container_cpu_usage_seconds_total{container!=""}[5m])))
/ sum(kube_pod_container_resource_requests{resource="cpu"}) * 100

# 99. Idle nodes (very low CPU utilisation — autoscaler down-scale candidates)
100 * (1 - avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[30m]))) < 5

# 100. Pods with memory requests far above actual usage (over-provisioning)
(kube_pod_container_resource_requests{resource="memory"} - on(pod) group_left() sum by (pod) (container_memory_working_set_bytes))
/ kube_pod_container_resource_requests{resource="memory"} > 0.7

# 101. Bonus: total cluster CPU capacity vs. total requested (headroom check)
sum(kube_node_status_allocatable{resource="cpu"}) - sum(kube_pod_container_resource_requests{resource="cpu"})
```

### 2.14 Query Optimization Techniques

You now have 100+ queries — the last piece is knowing how to keep them fast as your cluster and metric volume grow:

1. **Push expensive aggregations into Recording Rules (Chapter 7).** Any query you run repeatedly (every dashboard load, every alert evaluation) — like queries #59–64 above — should be a Recording Rule, not recomputed from raw data every time.
2. **Filter as early as possible in the expression.** `sum(rate(metric{namespace="prod"}[5m]))` is cheaper than `sum(rate(metric[5m]) and on(namespace) something)` — apply label matchers directly in the selector, not as a later filtering step, whenever possible.
3. **Avoid unnecessary regex matchers.** `{status="500"}` is cheaper than `{status=~"500"}` — use exact-match `=` whenever you're not actually relying on regex alternation/wildcarding.
4. **Be deliberate about range window size.** A `[5m]` `rate()` re-reads roughly 5 minutes of raw samples for every series involved; extremely wide windows (`[7d]`) on high-cardinality raw metrics, run ad hoc rather than as a Recording Rule, are a common source of slow, resource-heavy queries — this is exactly why query #60 (28-day SLI) should always be backed by a Recording Rule, never run raw in a dashboard.
5. **Watch `prometheus_engine_query_duration_seconds`** (Prometheus's own metric about its own query performance) to catch slow queries in your own dashboards before users notice a sluggish Grafana.
6. **Limit `topk`/`bottomk` and high-cardinality `by (...)` grouping in frequently-loaded dashboards** — grouping by a high-cardinality label (like `pod`) across a whole cluster in a query that runs on every dashboard refresh scales poorly; consider whether a coarser grouping (namespace) plus drill-down is a better UX and a cheaper query.

---

## 3. Why This Matters

This library is precisely the kind of reference document real platform teams maintain internally, and it directly operationalizes every concept from Parts 1–4: Golden Signals/RED/USE (Chapter 2) become queries 11–37 and 52–58; SLOs/error budgets (Chapter 3) become queries 59–64; the whole monitoring architecture (Chapter 4) is what makes queries 71–77 possible; and metric types (Chapter 6) are why every Counter query above is wrapped in `rate()`/`increase()` while every Gauge is read more directly.

---

## 4. Architecture

Not applicable — this chapter is a reference library, consumed by every later part: Part 6 (cAdvisor) expands section 2.3, Part 7 (Node Exporter) expands 2.2/2.12, Part 8 (kube-state-metrics) expands 2.4, Part 11 (Grafana) turns these into dashboard panels, Part 12 (Alerting) turns the SLO queries (2.6) into real Alertmanager-routed alerts, and Part 13 (Recording Rules) turns the expensive ones into pre-computed metrics per the optimization guidance in 2.14.

---

## 5. Hands-on Lab

Pick 10 queries from different sections above and run each against your Chapter 5 cluster (`http://localhost:9090/graph`, via the same port-forward as Chapter 8). For each one:

1. Run it in Table view first — confirm the label set and value make sense.
2. Switch to Graph view — confirm the shape over time looks sane (no unexplained gaps or spikes that don't match reality).
3. Note which ones return **no data yet** on your cluster (most of section 2.5's application RED queries — that's expected, since Online Boutique isn't deployed until Part 10) versus which return real data right now (nearly everything else, since it comes from Node Exporter, cAdvisor, kube-state-metrics, and the control plane, all already running from Chapter 5).

---

## 6. Verification

- [ ] Successfully run at least 10 queries from this chapter against your own cluster and correctly interpret the result.
- [ ] Explain why query #28 (CPU throttling) is a *saturation* signal and why it can be non-zero even when query #26/#27 (CPU usage/usage-vs-limit) look low — direct callback to Chapter 2's USE method.
- [ ] Explain why query #61 (error budget remaining) needs to be a Recording Rule in production rather than run raw on every dashboard load.
- [ ] Name at least 3 of the 6 optimization techniques from section 2.14 without looking.

---

## 7. Troubleshooting

| Symptom | Root Cause | Fix |
|---|---|---|
| Section 2.5/2.6 application queries return no data | Online Boutique isn't deployed/instrumented yet — that's Part 10. | Expected at this stage; revisit these queries after Part 10. |
| A query works in the Prometheus UI but times out or errors as a Grafana panel | Grafana may apply a different default time range or step, changing the effective range-vector window and query cost. | Check the panel's query inspector (Part 11) for the actual request sent; align the range window deliberately. |
| `predict_linear()` (queries #65/#66) gives an obviously wrong prediction | Insufficient or noisy historical data in the lookback window, or a genuine non-linear trend (linear extrapolation is a simple approximation, not a forecast model). | Use a longer, more stable lookback window; treat `predict_linear()` results as an early-warning heuristic, not a precise ETA. |
| A "governance check" query (e.g. #37, #51) returns more results than expected | These are `unless`/set-difference queries — small label mismatches between the two sides (e.g., extra labels on one side) can cause more/fewer matches than intuitively expected. | Verify both sides' label sets in Table view independently before combining with `unless`. |

---

## 8. Production Notes

- Real SRE teams don't memorize 100 queries — they maintain exactly this kind of **living, version-controlled reference document** (often alongside their Recording Rules, so the "cheap" version of an expensive query is right next to its raw definition) and teach new engineers to search it before writing a new query from scratch.
- Every query in section 2.6 (SLO/Error Budget) should, in a real production system, exist as a Recording Rule (Chapter 7, Part 13), not be run raw — they're included here in full, raw form specifically so you understand exactly what the resulting Recording Rule is actually computing under the hood.
- The optimization techniques in 2.14 compound — a dashboard with 20 panels, each independently running an unoptimized wide-window aggregation, is a very common real-world cause of a "why is my Grafana/Prometheus slow" ticket, and is directly solvable by moving the shared expensive parts into Recording Rules once (Part 13).

---

## 9. Best Practices

1. **Maintain your own internal query library**, organized by category exactly like this chapter, and keep it next to your Recording Rules definitions.
2. **Promote any query used in more than one dashboard or alert into a Recording Rule** — don't let the same expensive expression exist in five different places.
3. **Prefer exact-match label selectors over regex** wherever you're not actually using regex's power.
4. **Always sanity-check a new query in Table view before trusting its Graph rendering** or wiring it into an alert.
5. **Treat `predict_linear()`-based queries as early-warning heuristics**, and pair them with a human review step before fully automating any action off their output.

---

## 10. Interview Questions

1. **"Write a PromQL query to find the top 5 CPU-consuming pods in a cluster."** — `topk(5, sum by (pod, namespace) (rate(container_cpu_usage_seconds_total{container!=""}[5m])))`.
2. **"How would you detect that a container is being CPU-throttled even though its average usage looks fine?"** — Query `container_cpu_cfs_throttled_periods_total` relative to `container_cpu_cfs_periods_total` (query #28) — this is a saturation signal independent of average utilization, directly illustrating Chapter 2's USE method distinction.
3. **"How would you write an alert-ready query for SLO burn rate?"** — The ratio of the bad-event rate to the total-event rate, divided by (1 − SLO target), evaluated over both a short and long window (queries #62/#63) — the multi-window burn-rate pattern from Chapter 3, fully implemented in Part 12.
4. **"What's the difference between `predict_linear()` and just extrapolating by eye?"** — `predict_linear()` performs a proper linear regression over the specified lookback range and projects forward analytically; it's a systematic, queryable, alertable early-warning heuristic rather than an ad hoc visual guess — but it's still only a linear approximation and shouldn't be trusted for genuinely non-linear trends.
5. **"Give three concrete ways to make a slow PromQL dashboard faster."** — Push repeated expensive aggregations into Recording Rules, use exact-match instead of regex label matchers where possible, and avoid unnecessarily wide range windows or high-cardinality `by (...)` grouping in frequently-loaded panels (section 2.14).

---

## 11. Real Incident

**Company type:** Gaming platform with a large, actively-viewed Grafana dashboard suite (leadership kept several dashboards open on office wall displays).

**What happened:** A single shared "Executive Overview" dashboard had 40 panels, several of which independently computed the same expensive 28-day SLO compliance ratio (query #60-style) raw, on every page load/auto-refresh — with the dashboard set to auto-refresh every 30 seconds and displayed on 6 always-on wall monitors simultaneously. Prometheus's query load from this single dashboard alone was significant enough that it began measurably degrading query latency for on-call engineers trying to investigate an unrelated incident during a high-traffic period.

**Investigation:** `prometheus_engine_query_duration_seconds` (this chapter's optimization tip #5) showed a cluster of unusually slow, repeating queries; cross-referencing Grafana's dashboard JSON models (Part 11) against Prometheus's query log identified the Executive Overview dashboard as the source — 40 panels × 6 wall displays × every 30 seconds, several of them re-scanning 28 days of raw data each time.

**Resolution:** Converted every expensive shared aggregation on that dashboard into a Recording Rule (Chapter 7), computed once centrally regardless of how many times or how many places it was displayed; reduced the wall-display refresh interval to 5 minutes (real executive-dashboard needs rarely require 30-second freshness); query load and latency returned to baseline immediately.

**Prevention:** The team added "does this dashboard's underlying queries need to be Recording Rules" as a required review question before any dashboard could be added to the "always-on wall display" list — treating high-visibility, high-refresh-rate dashboards as a special, higher-scrutiny category rather than an afterthought.

---

## 12. Summary

- This chapter is a categorized, 100+ query reference library spanning cluster health, node/container resources (USE), Kubernetes object state, application RED metrics, SLO/error-budget math, capacity planning, control-plane health, alerting support, storage, debugging one-liners, and cost/efficiency.
- Every query builds directly on Parts 1–4's concepts — Golden Signals/RED/USE, SLIs/SLOs, the metric-journey architecture, and Counter/Gauge/Histogram mechanics.
- Query optimization (section 2.14) — Recording Rules for repeated expensive queries, exact-match over regex, careful range-window sizing, and cardinality-aware grouping — is what keeps a large real dashboard suite fast as a cluster scales.

---

## 13. Next Chapter

This closes out **Part 5: PromQL Masterclass.** You now have both the language fluency (Chapter 8) and a large working reference library (this chapter) to write essentially any query you'll need for the rest of this handbook.

**Part 6, Chapter 10: Container Metrics with cAdvisor** goes deep into exactly where queries like #26–#37 come from — cAdvisor's collection mechanism, every metric it exposes, and hands-on troubleshooting of real container resource problems (throttling, OOM, filesystem pressure) using the queries from this chapter as your working toolkit.
