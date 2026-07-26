# The Kubernetes Monitoring Handbook

**From zero to Production SRE — a complete, hands-on engineering playbook for building, operating, and troubleshooting production-grade Kubernetes monitoring platforms.**

This is not a blog series and not a set of course notes. It is written as an internal
platform engineering handbook, the kind a Fortune 100 company would hand to a new
Site Reliability Engineer on their first day on the monitoring team. Every chapter
builds on the one before it. Nothing is skipped. Nothing assumes prior knowledge
beyond Docker basics, Kubernetes basics, and `kubectl` basics.

## How this handbook is organized

The handbook is split into 20 parts. Each part contains one or more chapters.
Every chapter follows the same 13-section structure so you always know what to
expect: Objective, Concept, Why This Matters, Architecture, Hands-on Lab,
Verification, Troubleshooting, Production Notes, Best Practices, Interview
Questions, Real Incident, Summary, Next Chapter.

Throughout the labs we expose Grafana, Prometheus, and Alertmanager using
**NodePort only** — no cloud load balancers, no MetalLB, no Ingress. This keeps
the labs runnable on Kind, Minikube, kubeadm, bare metal, or any managed cloud
cluster (EKS/AKS/GKE/RKE2) without changing a single YAML file. Part 18 has a
dedicated chapter on LoadBalancer, Ingress, and Gateway API for when you're
ready to move beyond NodePort.

The reference application used throughout the hands-on labs is **Google's
Online Boutique** (formerly "microservices-demo") — a real, production-shaped
11-service polyglot microservices app (Go, Java, Python, Node.js, C#) with a
frontend, backend services, Redis cache, gRPC internal traffic, and load
generation. Later chapters extend it with a Postgres StatefulSet, a CronJob,
and a Job to round out the full set of Kubernetes workload types.

## Status

This handbook is being written sequentially, one chapter at a time, and this
index is updated as each chapter is completed. A checked box means the chapter
is done and merged; an unchecked box is planned but not yet written.

## Table of Contents

### Part 1 — Monitoring Fundamentals
- [x] [Chapter 1: What Is Monitoring? Metrics, Logs, Traces, and Observability](part-01-monitoring-fundamentals/chapter-01-what-is-monitoring-and-observability.md)
- [x] [Chapter 2: SRE, Golden Signals, RED, and USE](part-01-monitoring-fundamentals/chapter-02-sre-golden-signals-red-use.md)
- [x] [Chapter 3: SLIs, SLOs, and Error Budgets](part-01-monitoring-fundamentals/chapter-03-sli-slo-error-budgets.md)

### Part 2 — Monitoring Architecture
- [x] [Chapter 4: The Kubernetes Monitoring Stack — How Every Piece Fits Together](part-02-monitoring-architecture/chapter-04-kubernetes-monitoring-stack-architecture.md)

### Part 3 — Install Production Monitoring
- [x] [Chapter 5: Installing kube-prometheus-stack with Helm (NodePort)](part-03-install-production-monitoring/chapter-05-installing-kube-prometheus-stack.md)

### Part 4 — Understanding Prometheus
- [x] [Chapter 6: Prometheus TSDB, Labels, and Metric Types](part-04-understanding-prometheus/chapter-06-tsdb-labels-metric-types.md)
- [x] [Chapter 7: Recording Rules and Alert Rules](part-04-understanding-prometheus/chapter-07-recording-rules-and-alert-rules.md)

### Part 5 — PromQL Masterclass
- [x] [Chapter 8: PromQL Fundamentals](part-05-promql-masterclass/chapter-08-promql-fundamentals.md)
- [x] [Chapter 9: PromQL Advanced — 100+ Production Queries](part-05-promql-masterclass/chapter-09-promql-100-production-queries.md)

### Part 6 — cAdvisor Deep Dive
- [x] [Chapter 10: Container Metrics with cAdvisor](part-06-cadvisor-deep-dive/chapter-10-cadvisor-container-metrics.md)

### Part 7 — Node Exporter
- [x] [Chapter 11: Node Exporter and Linux Internals](part-07-node-exporter/chapter-11-node-exporter-linux-internals.md)

### Part 8 — kube-state-metrics
- [x] [Chapter 12: kube-state-metrics — Every Kubernetes Object Metric](part-08-kube-state-metrics/chapter-12-kube-state-metrics.md)

### Part 9 — Service Discovery
- [x] [Chapter 13: Service Discovery, ServiceMonitor, PodMonitor, Relabeling](part-09-service-discovery/chapter-13-service-discovery-relabeling.md)

### Part 10 — Monitoring Applications
- [x] [Chapter 14: Deploying and Instrumenting Online Boutique](part-10-monitoring-applications/chapter-14-deploying-instrumenting-online-boutique.md)

### Part 11 — Grafana
- [x] [Chapter 15: Grafana Mastery](part-11-grafana/chapter-15-grafana-mastery.md)

### Part 12 — Alerting
- [x] [Chapter 16: Alertmanager — Routing, Grouping, Silencing, Integrations](part-12-alerting/chapter-16-alertmanager.md)

### Part 13 — Recording Rules
- [x] [Chapter 17: Recording Rules in Depth](part-13-recording-rules/chapter-17-recording-rules-in-depth.md)

### Part 14 — OpenTelemetry
- [x] [Chapter 18: OpenTelemetry Collector — Metrics, Logs, Traces](part-14-opentelemetry/chapter-18-opentelemetry-collector.md)

### Part 15 — Loki
- [x] [Chapter 19: Loki and Promtail — Logging](part-15-loki/chapter-19-loki-promtail-logging.md)

### Part 16 — Tempo
- [x] [Chapter 20: Tempo — Distributed Tracing](part-16-tempo/chapter-20-tempo-distributed-tracing.md)

### Part 17 — Thanos
- [x] [Chapter 21: Thanos — HA, Long-Term Storage, Federation](part-17-thanos/chapter-21-thanos-ha-long-term-storage.md)

### Part 18 — Production Operations
- [x] [Chapter 22: Backup, Restore, Upgrade, Scaling, Capacity Planning](part-18-production-operations/chapter-22-backup-restore-upgrade-scaling-capacity.md)
- [x] [Chapter 23: Security — RBAC, TLS, NetworkPolicy](part-18-production-operations/chapter-23-security-rbac-tls-networkpolicy.md)
- [x] [Chapter 24: Beyond NodePort — LoadBalancer, Ingress, Gateway API](part-18-production-operations/chapter-24-beyond-nodeport-ingress-gateway-api.md)

### Part 19 — Production Troubleshooting
- [x] [Chapter 25: 100+ Real Incidents Runbook](part-19-production-troubleshooting/chapter-25-100-real-incidents-runbook.md)

### Part 20 — Interview Masterclass
- [x] [Chapter 26: 200+ Production Interview Questions](part-20-interview-masterclass/chapter-26-200-interview-questions.md)

---

**Status: COMPLETE.** All 20 parts, 26 chapters, written and cross-linked.

---

*Reference application:* [Online Boutique (Google Cloud microservices-demo)](https://github.com/GoogleCloudPlatform/microservices-demo)
