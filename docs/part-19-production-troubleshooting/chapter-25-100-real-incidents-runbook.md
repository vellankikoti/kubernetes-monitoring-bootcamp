# Chapter 25: 100+ Real Incidents Runbook

> **Part 19 — Production Troubleshooting**

---

## 1. Objective

This chapter is different from every other chapter in this handbook: it is a **reference runbook**, meant to be kept open during a real incident, not read cover to cover in one sitting. By the end of working through it you will be able to:

- Recognize the symptom patterns of 100+ real, common Kubernetes and monitoring-stack incidents.
- Know the exact diagnostic commands and PromQL queries to run for each, most of which you've already learned in Parts 1–18.
- Apply the correct resolution and, more importantly, the correct **prevention** — the difference between fixing an incident once and never seeing it again.

Every incident below is organized by category and follows the same compact shape: **Symptom → Root Cause → Investigation → Resolution → Prevention**. Three incidents per major category are expanded into full narrative walkthroughs (matching this handbook's usual "Real Incident" depth); the rest are presented as a dense reference table so this chapter is actually scannable under pressure, which is the whole point of a runbook.

---

## 2. Concept — The Runbook

### 2.1 Pod & Container Lifecycle Incidents

**Deep dive — Incident #1: CrashLoopBackOff on a newly deployed service**

*Symptoms:* `kubectl get pods` shows a pod cycling `Running → Error → CrashLoopBackOff`, with a climbing `RESTARTS` count.
*Investigation:* `kubectl logs <pod> --previous` (the *previous* container's logs, since the current one may not have logged anything useful yet); `kubectl describe pod <pod>` for the Events section and exit code.
*Root cause (most common):* application-level startup failure (bad config, missing env var, failed dependency connection) — exit code 1; OR a failed liveness probe repeatedly killing an otherwise-healthy-but-slow-starting container — exit code 137 with `Reason: Error` from a probe, not the app itself.
*Resolution:* Fix the underlying config/dependency issue; or adjust `initialDelaySeconds`/`failureThreshold` on the liveness probe if it's killing a legitimately slow starter.
*Prevention:* Always test a new deployment's liveness/readiness probe timing against real (not idealized) startup time before shipping; use `kube_pod_container_status_waiting_reason{reason="CrashLoopBackOff"}` (Chapter 12) as a standing alert.

**Deep dive — Incident #2: OOMKilled during a traffic spike**

*Symptoms:* Pod terminates suddenly during high load; `kubectl describe pod` shows `Last State: Terminated, Reason: OOMKilled, Exit Code: 137`.
*Investigation:* `container_memory_working_set_bytes` (Chapter 10) trended leading up to the kill time; compare against `kube_pod_container_resource_limits{resource="memory"}` (Chapter 12).
*Root cause:* Memory limit set below genuine peak usage under load — either the limit was undersized, or the application has a real memory leak that only manifests under sustained load.
*Resolution:* If usage was a legitimate, bounded peak: raise the memory limit (informed by Goldilocks/VPA data, Chapter 22). If usage climbed without bound over time: this is a leak — requires an application-level fix, not a limit increase.
*Prevention:* Load-test new services at realistic peak traffic before production rollout; alert on `container_memory_working_set_bytes / limit > 0.85` (Chapter 9, query #30) well before the actual OOM threshold.

**Deep dive — Incident #3: ImagePullBackOff after a registry credential rotation**

*Symptoms:* Pod stuck in `ImagePullBackOff`/`ErrImagePull`; `kubectl describe pod` shows `Failed to pull image: unauthorized`.
*Investigation:* `kubectl get secret <image-pull-secret> -n <namespace> -o yaml` to check existence/age; `kubectl describe pod` for the exact registry error.
*Root cause:* A registry credential (imagePullSecret) rotated/expired without the corresponding Kubernetes Secret being updated in every namespace that references it.
*Resolution:* Update/recreate the `imagePullSecret` with current credentials in the affected namespace(s).
*Prevention:* Automate imagePullSecret rotation and distribution (e.g., via an External Secrets Operator syncing from a central vault) rather than manual per-namespace updates; alert on `kube_pod_container_status_waiting_reason{reason="ImagePullBackOff"}` (Chapter 12).

| # | Symptom | Root Cause | Investigation | Resolution | Prevention |
|---|---|---|---|---|---|
| 4 | Pod `Pending` indefinitely | Insufficient cluster resources (no node has enough allocatable CPU/memory) | `kubectl describe pod` Events; `kube_node_status_allocatable` vs requests (Ch.9 #101) | Scale cluster/add nodes, or reduce requests | Capacity planning (Ch.22); cluster autoscaler |
| 5 | Pod `Pending`, "node affinity/anti-affinity" reason | No node matches required node selector/affinity rules | `kubectl describe pod`; check node labels | Fix affinity rules or label nodes correctly | Test affinity rules in staging before production |
| 6 | Pod `Pending`, taint-related | No node tolerates the pod, or pod lacks required toleration | `kubectl describe node` for taints; `kubectl describe pod` for tolerations | Add correct toleration or remove unneeded taint | Document taint/toleration conventions org-wide |
| 7 | Container stuck `Terminating` for a long time | `preStop` hook hanging, or `terminationGracePeriodSeconds` too long for a genuinely stuck process | `kubectl describe pod`; check preStop hook logic | Fix hook logic; consider `kubectl delete --grace-period=0 --force` as a last resort (data-loss risk) | Test graceful shutdown behavior explicitly before production |
| 8 | Sidecar container healthy, main container crashing, whole pod shows `NotReady` | Readiness probe on main container failing while sidecar's own probe passes | `kubectl describe pod` per-container status | Fix main container's readiness condition | Independent per-container dashboards, not just pod-level status |
| 9 | Init container stuck, app never starts | Init container waiting on a dependency (DB migration, config fetch) that's unavailable | `kubectl logs <pod> -c <init-container>` | Fix/restore the dependency the init container needs | Add explicit timeout + clear failure logging to init containers |
| 10 | Pod evicted with "The node was low on resource: ephemeral-storage" | Container's ephemeral filesystem usage exceeded available node capacity (Ch.10 §2.4) | `container_fs_usage_bytes` trend for the evicted pod | Reduce ephemeral writes (logs/temp files) or set explicit `ephemeral-storage` limits | Move heavy writes to a PVC; monitor ephemeral fs usage proactively |
| 11 | Pod restarts silently, no CrashLoopBackOff, restart count just climbs | Liveness probe failing intermittently under load (probe timeout too aggressive) | `kube_pod_container_status_restarts_total` (Ch.12) trend; correlate with load | Loosen probe timeout/threshold, or fix the underlying slow-response cause | Load-test probe behavior, not just happy-path startup |
| 12 | DaemonSet pod missing on one specific node | Node has a taint the DaemonSet doesn't tolerate, or resource pressure prevents scheduling | `kube_daemonset_status_desired_number_scheduled` vs `current` (Ch.12 §2.4) | Add toleration, or free up node resources | Alert on DaemonSet desired-vs-current mismatch |

### 2.2 Node-Level Incidents

**Deep dive — Incident #13: Node NotReady during a legitimate network blip**

*Symptoms:* `kubectl get nodes` shows a node `NotReady`; all pods on it eventually show as unreachable/unknown.
*Investigation:* `kube_node_status_condition{condition="Ready"}` (Ch.12); `node_systemd_unit_state{state="failed"}` (Ch.11) for kubelet itself; cloud provider status page for underlying infrastructure issues.
*Root cause:* Transient network partition between the node and control plane (kubelet heartbeat missed its threshold), not a genuine node failure.
*Resolution:* Usually self-resolves once network connectivity restores; if not, cordon/drain and investigate the node directly.
*Prevention:* Set appropriate `node-monitor-grace-period` for your environment's actual network reliability characteristics; alert on NotReady duration, not just occurrence (a 10-second blip is very different from a 10-minute one).

| # | Symptom | Root Cause | Investigation | Resolution | Prevention |
|---|---|---|---|---|---|
| 14 | Node `DiskPressure` condition | Node root filesystem usage exceeded eviction threshold | `node_filesystem_avail_bytes` (Ch.11); `kube_node_status_condition{condition="DiskPressure"}` (Ch.12) | Clean up unused images/logs; expand disk | Alert on filesystem usage trend before threshold is hit (Ch.9 #17) |
| 15 | Node `MemoryPressure` condition | Aggregate pod memory usage on the node exceeded safe threshold | `node_memory_MemAvailable_bytes` (Ch.11) | Evict/reschedule pods; rightsize workloads (Goldilocks, Ch.22) | Proper resource requests prevent overcommitment |
| 16 | Node cordoned and forgotten after maintenance | Human error — `kubectl uncordon` step skipped after planned work | `kube_node_spec_unschedulable` (Ch.12 §2.10) | `kubectl uncordon <node>` | Maintenance runbook checklist; alert on unexpectedly-long cordon duration |
| 17 | High CPU steal time on a specific node | Cloud "noisy neighbor" — hypervisor starving this VM's CPU | `rate(node_cpu_seconds_total{mode="steal"}[5m])` (Ch.11 §2.2) | Contact cloud provider support; consider dedicated/reserved instance type | Standing alert on sustained non-zero steal time |
| 18 | Node inexplicably slow, high load average, low CPU usage | Processes stuck in uninterruptible sleep (disk I/O wait) | `node_load1` vs core count (Ch.9 #13); `node_cpu_seconds_total{mode="iowait"}` | Investigate disk subsystem health directly | Disk I/O saturation alerting (Ch.9 #18) |
| 19 | Node runs out of available pod IPs | CNI IP address pool exhausted for that node/subnet | CNI-specific metrics/logs; `kubectl describe node` | Expand the CNI's IP pool/subnet allocation | Capacity planning for IP address space, not just CPU/memory |
| 20 | Node clock drift causing TLS/cert validation failures | NTP sync broken on the node | `node_timex_offset_seconds` (Ch.11 §2.7) | Restore NTP sync | Alert on clock offset exceeding a small threshold |
| 21 | containerd/CRI-O crashed, no new containers starting on that node | Underlying container runtime service failure (Ch.11's real incident) | `node_systemd_unit_state{state="failed"}` (Ch.11 §2.6) | Restart the runtime service | Standing cluster-wide alert on any failed systemd unit |

### 2.3 Networking & DNS Incidents

**Deep dive — Incident #22: Intermittent DNS resolution failures cluster-wide**

*Symptoms:* Random, intermittent "could not resolve host" errors across multiple unrelated services, not consistently reproducible.
*Investigation:* `sum(rate(coredns_dns_responses_total{rcode!="NOERROR"}[5m]))` (Ch.9 #75); CoreDNS pod resource usage and restart count; `coredns_dns_request_duration_seconds_bucket` (Ch.9 #76) for latency.
*Root cause:* CoreDNS pods under-provisioned (CPU-throttled, Ch.10) for actual cluster-wide DNS query volume, causing occasional query drops/timeouts under load.
*Resolution:* Increase CoreDNS replica count and/or resource limits; consider NodeLocal DNSCache to reduce load on central CoreDNS pods.
*Prevention:* Treat CoreDNS as a critical-path service with its own RED dashboard and capacity planning, not an invisible cluster default.

| # | Symptom | Root Cause | Investigation | Resolution | Prevention |
|---|---|---|---|---|---|
| 23 | Service unreachable from one specific namespace only | NetworkPolicy blocking the traffic (Ch.23) | Review NetworkPolicies in both namespaces | Add correct allow rule | Test NetworkPolicy changes against real cross-namespace traffic before applying broadly |
| 24 | ServiceMonitor target `DOWN`, connection refused | Wrong port name/number, or app not actually listening on expected port | `kubectl exec` debug pod, `curl` target directly (Ch.13 troubleshooting) | Fix port config mismatch | Verify new ServiceMonitors immediately after deploy (Ch.14 best practice) |
| 25 | Intermittent connection resets under high connection churn | TIME_WAIT / ephemeral port exhaustion (Ch.11 §2.5) | `node_sockstat_TCP_tw` trend | Connection pooling in app; kernel tuning | Monitor TIME_WAIT trend on connection-heavy services |
| 26 | East-west traffic mysteriously slow between two specific pods | Both scheduled on nodes with a degraded network path (rare hardware/switch issue) | `node_network_receive_errs_total`/`drop_total` (Ch.11 §2.5) on both nodes | Cordon/investigate the affected node's networking hardware | Standing network error/drop alerting per node |
| 27 | External LoadBalancer health checks failing intermittently | Readiness probe flapping, removing/re-adding the pod from the Service's endpoints repeatedly | `kube_pod_status_ready` (Ch.12) trend | Fix underlying readiness flap cause | Alert on readiness flapping frequency, not just current state |
| 28 | Ingress returns 502/504 under load | Backend Service's pods overwhelmed/not scaling fast enough | RED dashboard (Ch.15) for the backend service; HPA status (Ch.12 §2.9) | Scale up manually; check HPA maxReplicas ceiling (Ch.12 real incident) | Standing alert on HPA-at-max-replicas |
| 29 | Prometheus can't reach kubelet for cAdvisor scraping | Firewall/security group blocking port 10250 between Prometheus and nodes | `up{job="kubelet"}` == 0 (Ch.9 #1) | Open the required port in firewall/security group rules | Document required ports as part of cluster provisioning checklist |

### 2.4 Storage & PVC Incidents

**Deep dive — Incident #30: PVC stuck Pending, pod can't start**

*Symptoms:* New pod requiring a PVC stuck `Pending`; `kubectl get pvc` shows the claim also `Pending`.
*Investigation:* `kube_persistentvolumeclaim_status_phase{phase="Pending"}` (Ch.9 #43); `kubectl describe pvc` for the binding failure reason; `kubectl get storageclass` for a valid default class.
*Root cause:* No default StorageClass configured, or the requested storage class doesn't support dynamic provisioning in this environment (a common gap on some self-managed/bare-metal clusters, Ch.5's original storage note).
*Resolution:* Configure a working default StorageClass, or explicitly specify a valid one in the PVC spec.
*Prevention:* Verify StorageClass availability as part of initial cluster setup validation (Ch.5's own Verification checklist); document the cluster's supported storage classes.

| # | Symptom | Root Cause | Investigation | Resolution | Prevention |
|---|---|---|---|---|---|
| 31 | PVC usage approaching 100%, application errors on write | Under-provisioned volume size for actual data growth | `kubelet_volume_stats_used_bytes / capacity_bytes` (Ch.9 #82) | Expand the PVC (if StorageClass supports online resize) | Alert at 85% PVC usage, well before the actual failure threshold |
| 32 | "No space left on device" despite PVC showing free bytes | Inode exhaustion, not byte exhaustion (Ch.11 §2.4) | `node_filesystem_files_free` / `kubelet_volume_stats_inodes_used` (Ch.9 #83) | Clean up excessive small files | Monitor inode usage alongside byte usage, always |
| 33 | Prometheus data loss after pod reschedule | No `storageSpec`/PVC configured (Ch.5's real incident, verbatim) | `kubectl get pvc -n monitoring` empty | Reinstall/reconfigure with explicit persistent storage | Always verify PVC exists as part of any stateful Helm install's checklist |
| 34 | StatefulSet pod stuck, volume "multi-attach" error | PVC with `ReadWriteOnce` access mode being claimed by a pod on a different node than the old one | `kubectl describe pod` Events | Wait for old pod's volume detach, or manually force-delete the stuck old pod if truly orphaned | Understand access mode implications before choosing StorageClass |
| 35 | Orphaned PersistentVolumes accumulating, wasting cost | PVs left in `Released` phase after PVC deletion (reclaim policy `Retain`) | `kube_persistentvolume_status_phase{phase="Released"}` (Ch.9 #85-adjacent) | Manually review and delete/reclaim as appropriate | Deliberate reclaim policy choice per data sensitivity; periodic orphan review |
| 36 | Postgres StatefulSet (Ch.14) pod restart loses recent writes | Missing/misconfigured PVC, or the app itself not using the mounted volume correctly for its data directory | Check StatefulSet's `volumeMounts` and `volumeClaimTemplates` | Correct the volume mount path to match the app's actual data directory | Verify data persistence explicitly with a restart test before trusting a new stateful deployment |

### 2.5 Prometheus-Specific Incidents

**Deep dive — Incident #37: Prometheus OOMKilled from a cardinality explosion**

*Symptoms:* Prometheus pod repeatedly OOMKilled; cluster-wide monitoring degraded or unavailable during the crash loop.
*Investigation:* Once briefly stable, check `Status → TSDB Status` (Ch.6 §2.4 lab); `prometheus_tsdb_head_series` trend (Ch.6, Ch.9 #8).
*Root cause:* An unbounded-cardinality label (user ID, request ID) added to a metric (Ch.6's real incident, verbatim).
*Resolution:* Remove the offending label from instrumentation; add a `metric_relabel_configs` drop rule as an immediate mitigation (Ch.13).
*Prevention:* Cardinality-lint new metrics in CI/code review; standing alert on `prometheus_tsdb_head_series` growth rate.

| # | Symptom | Root Cause | Investigation | Resolution | Prevention |
|---|---|---|---|---|---|
| 38 | Prometheus `up{job=...}==0` for a specific target | Target genuinely down, network issue, or wrong port/label selector (Ch.13's full troubleshooting table) | Service Discovery UI page (Ch.13) | Depends on root cause found | Verify new ServiceMonitors immediately after deploy |
| 39 | PromQL query times out in Grafana | Unoptimized, unrecorded expensive query run repeatedly (Ch.9's Executive Dashboard incident) | `prometheus_engine_query_duration_seconds` | Convert to a Recording Rule (Ch.17) | Apply the "repeated AND expensive" decision framework (Ch.17 §2.2) proactively |
| 40 | Alert never fires despite condition clearly true | `PrometheusRule` label selector mismatch with `Prometheus` CRD's `ruleSelector` (Ch.7's most common gotcha) | Compare labels directly | Fix the label to match | Standardize labeling via a shared Helm template/policy |
| 41 | Rule group evaluation falling behind its own interval | Too many expensive unrecorded rules crammed into one group (Ch.17 §2.6) | `prometheus_rule_group_last_duration_seconds` > `interval_seconds` | Split groups; push work into lower recording-rule layers | Monitor rule group evaluation duration as a standing health check |
| 42 | Old metric data missing despite retention looking sufficient | Disk-based retention cap (`retention.size`) triggered early eviction | Check both `retention.time` and `retention.size` config | Increase PVC size or reduce cardinality/scrape frequency | Understand both retention dimensions, not just the time-based one |
| 43 | Prometheus slow to become Ready after restart | WAL replay proportional to unflushed head-block data at crash time (Ch.6 §2.4) | Prometheus startup logs | Usually self-resolves; ensure adequate CPU for faster replay | Reduce cardinality/scrape volume if replay time becomes operationally painful |
| 44 | `histogram_quantile()` returns NaN or clearly wrong values | Missing `by (le)` in the inner aggregation (Ch.8's real incident, verbatim) | Manually inspect the query structure | Use the canonical pattern: `histogram_quantile(q, sum by (le) (rate(...)))` | PromQL review checklist requiring the canonical histogram pattern |
| 45 | Two Prometheus HA replicas showing different values for "the same" query | Expected — independent scrape timing jitter between replicas; no deduplication configured (Ch.4 §2.6, Ch.21) | Compare each replica directly via its own `replica` label | Deploy Thanos Query for proper deduplication | Understand HA replica behavior before assuming a bug |
| 46 | Remote-write from OTel Collector not reaching Prometheus | `enableRemoteWriteReceiver` not set (Ch.18 troubleshooting) | Check Prometheus CRD config | Enable the remote-write receiver flag | Include in OTel Collector deployment checklist |
| 47 | Prometheus CPU usage spikes precisely every 2 hours | Normal head-block-to-disk-block compaction cycle (Ch.6 §2.4) | Correlate spike timing with block boundaries | Usually expected behavior, not a bug — verify resource headroom accounts for it | Size CPU limits with compaction cycles in mind, not just steady-state scrape load |

### 2.6 Grafana-Specific Incidents

| # | Symptom | Root Cause | Investigation | Resolution | Prevention |
|---|---|---|---|---|---|
| 48 | Panel shows "No Data" despite query working in Prometheus UI | Wrong data source selected, or variable substitution producing an invalid label value (Ch.15 troubleshooting) | Grafana Query Inspector | Fix data source/variable config | Test panels with the Query Inspector before publishing |
| 49 | Dashboard variable dropdown empty | Underlying `label_values()` query matches no real data | Test the raw query in Prometheus UI directly | Fix the variable's query or wait for real data to exist | Validate variable queries against real cluster data before sharing a dashboard |
| 50 | Provisioned dashboard ConfigMap never appears in Grafana | Missing/incorrect `grafana_dashboard` label (Ch.15 §2.7) | Grafana sidecar container logs | Add the correct label | Template dashboard ConfigMaps from a known-good example |
| 51 | Edits to a dashboard disappear after pod restart | Dashboard is provisioned (file-based), UI edits don't persist by design (Ch.15, deliberate behavior) | N/A — expected behavior | Make the change in the source JSON/ConfigMap instead | Document this behavior clearly for new team members |
| 52 | Heatmap panel renders as meaningless colored blocks | "Format" not set to Heatmap, or bucket boundaries too sparse (Ch.15 §2.6, Ch.14 §2.4) | Panel edit → Format setting | Set Format to Heatmap; revisit bucket boundaries | Standardize heatmap panel setup in dashboard templates |
| 53 | Grafana login fails after Secret-based admin migration | Wrong `userKey`/`passwordKey` reference, or Secret in wrong namespace (Ch.23 troubleshooting) | `kubectl get secret -o yaml` | Correct the key names/namespace | Test login immediately after any credential migration |
| 54 | Exemplars not appearing on latency panels | Not enabled on the Prometheus data source, or client library doesn't attach exemplars (Ch.20 troubleshooting) | Grafana data source settings | Enable exemplars explicitly; verify library support | Include exemplar enablement in standard data source setup |
| 55 | Grafana dashboard extremely slow to load | Too many panels independently running unrecorded expensive queries (Ch.9's real incident) | Query Inspector timing per panel | Convert shared expensive queries to Recording Rules | Cap panel count per dashboard; recording-rule-back anything reused |

### 2.7 Alertmanager-Specific Incidents

| # | Symptom | Root Cause | Investigation | Resolution | Prevention |
|---|---|---|---|---|---|
| 56 | Alert firing in Prometheus but never notified | Silence or inhibition suppressing it unexpectedly (Ch.16 §2.5/2.6) | Alertmanager UI alert detail view | Review/adjust the silence or inhibition rule | Comment every silence with a reason/ticket; review inhibition rules carefully |
| 57 | One alert generates 30 separate notifications instead of one | `group_by` too granular (e.g., grouping by `pod` instead of `alertname`) (Ch.16 §2.2) | Review routing config | Broaden `group_by` to represent "the same incident" | Standardize `group_by` conventions org-wide |
| 58 | Alert should go to two channels but only reaches one | Missing `continue: true` on an earlier matching route (Ch.16 §2.3, common gotcha) | Alertmanager UI route-matching view | Add `continue: true` | Verify multi-receiver routing via the UI, don't assume config correctness |
| 59 | `AlertmanagerConfig` applied but has no effect | Label selector mismatch with `Alertmanager` CRD's `alertmanagerConfigSelector` (same pattern as Ch.7/13) | Compare labels | Fix the label | Standardize labeling via shared templates |
| 60 | Notification never repeats for a still-firing alert | `repeat_interval` set too long for the severity | Check alert's actual state in Alertmanager UI directly | Adjust `repeat_interval` appropriately for severity | Match `repeat_interval` to severity tier deliberately |
| 61 | Unrelated critical alert silently suppressed during an incident | Overly broad inhibition rule `equal` matching (Ch.16's real incident, verbatim) | Alertmanager UI, source alert visible in detail view | Tighten the inhibition rule's matching fields | Require a documented non-suppression test case for new inhibition rules |
| 62 | Slack notifications stop arriving entirely | Webhook URL expired/revoked, or Slack-side rate limiting | `rate(alertmanager_notifications_failed_total[5m])` (Ch.9 #81) | Regenerate/update the webhook URL | Standing alert on notification failure rate |

### 2.8 Application & Instrumentation Incidents

| # | Symptom | Root Cause | Investigation | Resolution | Prevention |
|---|---|---|---|---|---|
| 63 | Intermittent latency spikes, average CPU looks fine | CPU throttling (Ch.10's real incident, verbatim) | `container_cpu_cfs_throttled_periods_total` ratio (Ch.10 §2.2) | Raise CPU limit or fix application burstiness | Add throttling ratio as a required panel on latency-sensitive dashboards |
| 64 | Cluster-wide p99 dashboard looks suspiciously smooth after scaling out | Averaging a Summary's per-instance quantiles across pods (Ch.6 §2.5, mathematically invalid) | Check metric TYPE | Re-instrument with a Histogram | Prefer Histogram over Summary for anything aggregated across instances |
| 65 | New service has no metrics at all despite `/metrics` working via curl | ServiceMonitor missing or misconfigured (Ch.13 full troubleshooting table) | Service Discovery UI page | Deploy/fix the ServiceMonitor | Make ServiceMonitor deployment part of the standard service scaffold |
| 66 | Service's histogram-based latency dashboard looks coarse/inaccurate | Bucket boundaries don't match actual latency distribution (Ch.14 §2.4) | Compare buckets against real observed latencies | Tune bucket boundaries per-service | Set bucket boundaries deliberately at instrumentation time, not generic defaults |
| 67 | gRPC service error rate looks zero despite real failures | Querying HTTP-style `status` label instead of gRPC's `grpc_code` label | Check actual label names on the metric | Correct the query to use the right label | Document per-protocol RED query patterns (Ch.9 §2.5) |
| 68 | HPA not scaling despite genuine load | Already at `maxReplicas` ceiling (Ch.12's real incident, verbatim) | `current_replicas == spec_max_replicas` (Ch.9 #45) | Raise `maxReplicas` after confirming cluster capacity | Standing alert on HPA-at-max-replicas; periodic maxReplicas review |
| 69 | CronJob silently stops running | `spec.suspend: true` set accidentally, or bad cron syntax | `kube_cronjob_status_last_schedule_time` staleness (Ch.9 #47) | Fix suspend flag or schedule syntax | Alert on CronJob schedule staleness |
| 70 | Deployment rollout stuck, mix of old/new pods indefinitely | New ReplicaSet's pods failing readiness (resource constraint or bad config) | `kube_deployment_status_replicas_unavailable` (Ch.12 §2.2) + new pods' resource metrics | Fix the new pods' underlying issue | Canary/staged rollouts with automated rollback on readiness failure |

### 2.9 Security & Certificate Incidents

| # | Symptom | Root Cause | Investigation | Resolution | Prevention |
|---|---|---|---|---|---|
| 71 | Externally-facing service suddenly returns TLS errors | Certificate expired, manual renewal missed (Ch.24's real incident, verbatim) | `kubectl describe certificate` (if using cert-manager) | Renew certificate immediately | Automate renewal via cert-manager; alert on certificate expiry approaching |
| 72 | NetworkPolicy applied, legitimate traffic unexpectedly blocked | `podSelector`/`namespaceSelector` in the allow rule doesn't match real labels (Ch.23 troubleshooting) | `kubectl get pods --show-labels` | Correct the selector to match actual labels | Test NetworkPolicy changes in staging against real traffic first |
| 73 | Sensitive data discovered in log content during a security review | Application logging PII/tokens directly (Ch.23's real incident, verbatim) | Manual log content audit | Remove sensitive content from logging statements; restrict access in the meantime | Code review checklist item for any new logging statement |
| 74 | Unexpected cross-namespace metric scraping discovered | Missing `namespaceSelector` on a ServiceMonitor, label collision (Ch.13's real incident) | Service Discovery UI "Discovered Labels" | Add explicit `namespaceSelector` | Enforce explicit namespaceSelector via policy for all ServiceMonitors |
| 75 | RBAC audit finds a component with broader access than expected | Chart defaults erring toward broad-but-functional rather than minimal (Ch.23 §2.1) | `kubectl describe clusterrole` | Consider a tightened custom Role if required by security posture | Recurring RBAC audit as a scheduled practice, not one-time |

### 2.10 Cross-Cutting / "Everything Looks Fine But Isn't" Incidents

These are the hardest category — no single component is obviously broken, but the *system* is.

| # | Symptom | Root Cause | Investigation | Resolution | Prevention |
|---|---|---|---|---|---|
| 76 | RED dashboard fully green, but a queue-based consumer is falling behind | RED doesn't measure backlog/lag (Ch.2's real incident, verbatim) | Consumer group lag metric directly | Rebalance/scale the consumer; add lag as a first-class signal | Recognize when RED needs a domain-specific extension (Ch.2 §2.5) |
| 77 | SLO compliance report looks fine, but a real, sharp outage happened last week | Rolling-window SLO smoothing (Ch.3 §2.4) hides a short, sharp incident within a long compliance window | Compare burn-rate alert history against the compliance number | Use burn rate for real-time detection, compliance % only for trend reporting | Don't rely on SLO compliance alone for incident awareness |
| 78 | On-call ignores a real page because of alert fatigue | An alert's `for`/threshold was loosened without review, causing weeks of false positives first (Ch.7's real incident, verbatim) | Alert firing history | Restore correct threshold/`for`; rebuild on-call trust deliberately | Require review for any change to an existing alert's sensitivity |
| 79 | "We had monitoring but still had a 6-hour outage" | Monitoring (known unknowns) without observability (unknown unknowns) — Ch.1's core distinction, hit for real | Post-incident review of what data existed vs. what was needed | Invest in logs/traces correlation, not just more dashboards | Treat observability, not just monitoring, as the actual goal from the start |
| 80 | Capacity incident nobody saw coming despite "good monitoring" | Nobody applied `predict_linear()`-style trend analysis to the monitoring stack's own resource usage (Ch.22 §2.5) | `predict_linear()` on Prometheus's own metrics, retroactively | Fix the immediate capacity issue | Apply capacity planning to the platform itself, not just the workloads it watches |

### 2.11 Rapid-Reference Table — Remaining Common Incidents (81–105)

| # | Symptom | Root Cause (short) | First command/query to run |
|---|---|---|---|
| 81 | `kubectl top` shows nothing | Metrics Server down/misconfigured | `kubectl get deployment metrics-server -n kube-system` |
| 82 | Namespace stuck `Terminating` | Finalizer never completes (Ch.12 §2.5) | `kubectl get ns <name> -o yaml`, check `status.conditions` |
| 83 | Secret value appears in a Prometheus label | Accidental instrumentation of sensitive data as a label | Audit instrumentation code immediately |
| 84 | Thanos Query returns no data | Store endpoint unregistered/unhealthy (Ch.21 troubleshooting) | Thanos Query's `/stores` status page |
| 85 | Thanos Sidecar not uploading blocks | Object storage credentials wrong, or no finalized blocks yet | Sidecar container logs |
| 86 | Loki ingestion extremely slow | High-cardinality stream labels (Ch.19's real incident, verbatim) | Review Promtail relabeling config for unbounded labels |
| 87 | LogQL query times out | Stream selector too broad for the time range (Ch.19 troubleshooting) | Narrow with more specific labels first |
| 88 | Trace has "gaps," missing spans from one service | Broken context propagation at an uninstrumented hop (Ch.20 troubleshooting) | Audit that specific service-to-service call |
| 89 | TraceQL query returns nothing despite matching traces existing | Attribute name/casing mismatch (span vs resource level, Ch.20) | Double-check attribute scope |
| 90 | OTel Collector dropping data | `memory_limiter` shedding load under pressure (Ch.18's real incident, verbatim) | `otelcol_processor_dropped_metric_points_total` |
| 91 | Compactor-related object storage corruption | Multiple uncoordinated Compactor instances (Ch.21 §8, documented gotcha) | Confirm single Compactor instance / leader election |
| 92 | VPA/Goldilocks recommendations look wildly off | Insufficient observation history yet | Allow more time before trusting recommendations |
| 93 | OpenCost dollar figures look implausible | Node pricing config outdated/wrong for actual instance types | Verify OpenCost's pricing configuration against real billing |
| 94 | Helm upgrade "succeeds" but behavior doesn't change | CRDs not auto-upgraded by Helm (Ch.22's real incident, verbatim) | Check CRD version explicitly, apply new CRDs manually |
| 95 | Rollback doesn't fully restore previous behavior | CRDs not rolled back either (same asymmetry, reverse direction) | Manually verify/revert CRD state too |
| 96 | Ingress returns default backend 404 | Host header mismatch or Ingress rule typo (Ch.24 troubleshooting) | Verify DNS resolution and exact rule match independently |
| 97 | cert-manager Certificate stuck, never issues | DNS/HTTP challenge failing (no public DNS, or misconfigured issuer) | `kubectl describe challenge` |
| 98 | Gateway API HTTPRoute has no effect | No matching `Gateway` object, or controller lacks Gateway API support | Confirm `Gateway` object exists and controller version supports it |
| 99 | Two unrelated services intermittently cross-talk | NodePort collision (Ch.24's real incident, verbatim) | Audit NodePort assignments for duplicates |
| 100 | Postgres StatefulSet (Ch.14) loses data on restart | PVC misconfigured or mount path wrong | Verify `volumeClaimTemplates` and actual data directory mount |
| 101 | Job never completes, stuck `Active` | Underlying task hanging, no timeout configured | `kubectl logs` the Job's pod directly; add `activeDeadlineSeconds` |
| 102 | Same alert fires from two different unrelated root causes, confusing on-call | Alert's `expr`/annotations too generic to distinguish causes | Add more specific labels/annotations distinguishing likely causes |
| 103 | Dashboard shows data for the wrong time zone | Grafana/browser time zone setting mismatch, not a data problem | Check dashboard/user time zone settings first before doubting the data |
| 104 | Recording rule output has surprisingly high cardinality | `by (...)` clause preserved a high-cardinality label unintentionally (Ch.17 troubleshooting) | Review the rule's `by (...)` clause deliberately |
| 105 | New engineer can't reproduce a "known" incident from the runbook | Environment/version drift since the runbook entry was written | Verify the runbook entry still matches current architecture; update it |

---

## 3. Why This Matters

- This chapter is the payoff of every single concept taught in Parts 1–18 — nearly every entry above cites the specific earlier chapter where the underlying mechanism was actually taught, because a runbook entry that just says "fix it" without that grounding produces engineers who can follow a checklist but can't adapt when the *real* incident doesn't match any entry exactly.
- Real incidents are rarely identical to a runbook entry — the value of having internalized *why* each fix works (not just *that* it works) is precisely what lets you adapt an entry like #37 (cardinality explosion in Prometheus) to a genuinely novel variant you've never seen before, using the same underlying TSDB/cardinality mental model from Chapter 6.
- Several entries (76, 77, 78, 79, 80) are deliberately placed in a "cross-cutting" category specifically because they represent failures where *no single component was broken* — these are the hardest, most senior-level incidents, and recognizing this category as distinct from "component X failed" is itself a mark of real operational maturity.

---

## 4. Architecture

Not applicable in the usual sense — this chapter's "architecture" is its own organization: 10 categories mapping directly onto this handbook's Parts (Pod/Container → Part 6, Node → Part 7, Networking/DNS → Part 9, Storage → throughout, Prometheus → Parts 4–5, 13, 17, Grafana → Part 11, Alertmanager → Part 12, Application → Parts 2, 10, Security → Part 18, Cross-cutting → Parts 1–3).

---

## 5. Hands-on Lab

Pick 5 incidents from different categories above that you have **not** already deliberately reproduced in an earlier chapter's own hands-on lab, and reproduce them for real against your cluster:

- Deliberately misconfigure a `ServiceMonitor`'s label selector (incident #40) and confirm you can diagnose it using only the Service Discovery UI page, without looking at this chapter's answer first.
- Deliberately set an unreasonably low memory limit on a test deployment and trigger a real OOMKill (incident #2), then walk the full investigation using `container_memory_working_set_bytes`.
- Deliberately create a NodePort collision (incident #99) between two test Services and observe the resulting cross-talk directly.

This kind of **deliberate incident injection** (sometimes called chaos engineering in its more formalized, production-grade form) is a genuine, valuable practice — the muscle memory from diagnosing a problem you caused yourself, in a safe lab environment, transfers directly to diagnosing the same class of problem for real, under pressure, later.

---

## 6. Verification

- [ ] For at least 10 incidents across at least 5 different categories, you can state the symptom, root cause, and first diagnostic command from memory, without looking at the table.
- [ ] You can correctly categorize a novel, hypothetical incident description into the right category (Pod/Container, Node, Networking, Storage, Prometheus, Grafana, Alertmanager, Application, Security, or Cross-cutting) and identify which earlier chapter's concept it most likely relates to.
- [ ] You've successfully reproduced and diagnosed at least 3 incidents in your own lab cluster, per section 5.

---

## 7. Troubleshooting

*(This entire chapter is a troubleshooting reference — see section 2 above.)*

---

## 8. Production Notes

- Real SRE teams maintain a runbook exactly like this one, but **alive and versioned** — every genuinely new incident type encountered gets added as a new entry, and every entry that turns out to be wrong or outdated (architecture changes, tool upgrades) gets corrected, exactly as incident #105 above describes as its own category of failure.
- The **"three deep dives per category, rest as a table"** structure used in this chapter mirrors how real, mature internal runbooks are actually organized — full narrative depth for the most instructive/common cases, dense reference format for rapid lookup during an actual active incident, when nobody has time to read a full narrative.
- **Cross-cutting incidents (2.10) are systematically underrepresented in most teams' runbooks** compared to component-specific ones, precisely because they're harder to write a clean "symptom → root cause" entry for — worth deliberately over-investing in documenting these, since they're also often the most damaging and hardest to recognize in the moment.

---

## 9. Best Practices

1. **Keep this runbook (or your own organization's equivalent) genuinely alive** — add new incidents as they happen, correct stale entries, don't let it fossilize.
2. **Practice deliberate incident injection** in a safe lab environment (section 5) — real diagnostic muscle memory is built by doing, not just reading.
3. **Always ask "which category is this" first** during a real incident — it immediately narrows which earlier chapter's mental model to reach for.
4. **Give cross-cutting incidents (2.10) deliberate extra attention** in your own team's runbook — they're underrepresented by default and disproportionately damaging when missed.
5. **Cite the underlying mechanism, not just the fix**, in every runbook entry you write — a future engineer adapting your entry to a slightly different real incident needs the "why," not just the "what."

---

## 10. Interview Questions

1. **"Walk me through your diagnostic process when a pod is stuck in CrashLoopBackOff."** — Check `kubectl logs --previous` and `kubectl describe pod` for the exit code and Events; distinguish an application-level failure (bad config/dependency) from a probe-induced kill (liveness probe timing too aggressive for actual startup time).
2. **"A service's average CPU usage looks fine but users report intermittent slowness. What's your first hypothesis and how do you check it?"** — CPU throttling, independent of average usage (Ch.10); check the throttled-periods-to-total-periods ratio directly.
3. **"How do you distinguish a genuine memory leak from an undersized memory limit when investigating an OOMKill?"** — Check whether `container_memory_working_set_bytes` climbs without bound over time (leak) versus reaching a legitimate, bounded peak under load that simply exceeds the configured limit (sizing issue).
4. **"What's the difference between a component-specific incident and a 'cross-cutting' incident, and why does that distinction matter operationally?"** — Component-specific incidents have one clearly broken piece (a pod, a node, Prometheus itself); cross-cutting incidents (alert fatigue, SLO-window smoothing hiding a real outage, monitoring-without-observability) involve no single broken component — the system as a whole fails a purpose despite every individual piece technically "working," and these are typically the hardest and most senior-level incidents to recognize and prevent.
5. **"How do you keep an incident runbook actually useful over time, rather than letting it go stale?"** — Treat it as a living document: add every genuinely new incident type as it's encountered, and explicitly correct or retire entries once they no longer match the current architecture — a stale runbook (incident #105 in this very chapter) is itself a recognized failure mode.

---

## 11. Real Incident

*(This chapter's entire section 2 IS the real incident library — see the deep-dive entries throughout, each drawn from and cross-referencing this handbook's own earlier chapters' real incidents.)*

---

## 12. Summary

- This chapter consolidated 100+ real, categorized incidents spanning Pod/Container lifecycle, Node health, Networking/DNS, Storage, Prometheus internals, Grafana, Alertmanager, Application/instrumentation, Security, and cross-cutting systemic failures.
- Every entry traces back to a specific mechanism taught somewhere in Parts 1–18 — this runbook is not a new body of knowledge, it's an **organized, scannable index into everything already learned**, structured the way a real on-call engineer actually needs it: fast to search, with just enough context to adapt to a not-quite-identical real situation.
- **Cross-cutting incidents** — where no single component is broken but the system still fails its purpose — are the hardest category and deserve deliberate, disproportionate attention in any real team's own runbook.

---

## 13. Next Chapter

This closes out **Part 19: Production Troubleshooting.**

**Part 20, Chapter 26: 200+ Production Interview Questions** consolidates this entire handbook's knowledge into the final form most readers will use it for at some point in their career: a comprehensive, categorized interview preparation reference spanning architecture, PromQL, Grafana, Prometheus internals, cAdvisor, ServiceMonitor/service discovery, Alertmanager, OpenTelemetry, and incident response — the last chapter of the handbook, and the one that proves you can articulate, out loud, everything you've now built and operated hands-on.
