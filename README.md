# Kubernetes Monitoring Bootcamp

**The complete Kubernetes Monitoring Handbook — from zero to Production SRE.**

A hands-on engineering playbook for building, operating, and troubleshooting production-grade Kubernetes monitoring platforms with Prometheus, Grafana, Alertmanager, Loki, Tempo, and Thanos — written as an internal-style handbook, not a blog series.

- **20 parts, 26 chapters**, each following the same Objective → Concept → Why It Matters → Architecture → Hands-on Lab → Verification → Troubleshooting → Production Notes → Best Practices → Interview Questions → Real Incident → Summary structure.
- Real, runnable labs against **kube-prometheus-stack** (Helm, NodePort-only for portability across Kind/Minikube/kubeadm/bare metal/EKS/AKS/GKE/RKE2).
- A real reference application — Google's **Online Boutique** (11-service polyglot microservices demo), extended with a Postgres StatefulSet, a CronJob, and a Job to exercise every Kubernetes workload type.
- 100+ production PromQL queries, layered Recording Rules, full multi-window burn-rate SLO alerting, OpenTelemetry/Loki/Tempo/Thanos, a 100+ incident troubleshooting runbook, and 200+ interview questions.
- Every diagram is native Mermaid (flowcharts, state diagrams, sequence diagrams, and a Gantt-based trace waterfall).

## Start here

🌐 **[Live interactive guide](https://vellankikoti.github.io/kubernetes-monitoring-bootcamp/)** — sidebar navigation across all 26 chapters, rendered Mermaid diagrams, light/dark theme.

📖 **[docs/README.md](docs/README.md)** — the same content as plain Markdown, full table of contents, Part 1 through Part 20.

## Structure

```
docs/
├── part-01-monitoring-fundamentals/
├── part-02-monitoring-architecture/
├── ...
└── part-20-interview-masterclass/
```

Each part directory contains one or more chapter Markdown files. Start at Chapter 1 and work through in order — later chapters assume the concepts and running cluster built in earlier ones.
