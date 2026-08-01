# Zen Pharma — DevOps Interview Prep

> 125+ interview questions curated from real interviews at Google, Amazon, Microsoft, Netflix, Uber, Meta, JPMorgan, Goldman Sachs.
> All answers grounded in the zen-pharma project — answer every question with "In our project, we..."

---

## Question Index

| File | Topic | Questions |
|------|-------|-----------|
| [01-terraform.md](./01-terraform.md) | Terraform & IaC | 45 |
| [02-eks-kubernetes.md](./02-eks-kubernetes.md) | EKS & Kubernetes | 24 |
| [03-cicd-github-actions.md](./03-cicd-github-actions.md) | CI/CD & GitHub Actions | 18 |
| [04-gitops-argocd.md](./04-gitops-argocd.md) | GitOps & ArgoCD | 23 |
| [05-security-devsecops.md](./05-security-devsecops.md) | Security & DevSecOps | 24 |
| [06-vpc-networking-aws.md](./06-vpc-networking-aws.md) | VPC, Networking & AWS | 18 |
| [07-behavioral-realtime.md](./07-behavioral-realtime.md) | Behavioral & Real-Time | 19 |
| [08-observability-logging-monitoring.md](./08-observability-logging-monitoring.md) | Observability — Logging & Monitoring | 24 |
| [09-live-troubleshooting-labs.md](./09-live-troubleshooting-labs.md) | Live Troubleshooting Labs — break it, investigate it, fix it | 12 |
| **Total** | | **207** |

---

## Project Stack

| Layer | Technology |
|-------|-----------|
| IaC | Terraform 1.10, S3 backend with native locking |
| Cloud | AWS us-east-1: EKS 1.33, RDS PostgreSQL 15.7, ECR, Secrets Manager |
| Kubernetes | NGINX Ingress, External Secrets Operator, Kyverno, Prometheus/Grafana |
| CI/CD | GitHub Actions (reusable workflows), GitHub OIDC, ArgoCD |
| Security | CodeQL, Semgrep, OWASP Dep Check, Trivy, Cosign/Sigstore |
| GitOps | zen-gitops → Helm charts → ArgoCD → EKS namespaces (dev/qa/prod) |
| Observability | Fluent Bit (DaemonSet) → Elasticsearch (Elastic Cloud); kube-prometheus-stack (Prometheus/Grafana/Alertmanager) |

---

## Interview Tips

1. **Always use project context.** "In our zen-pharma project, we do X because Y" beats a theoretical answer every time.
2. **Know the flow end-to-end.** The most common opener: "Walk me through how code gets to production." Practice this until it's 90 seconds, fluent, with specific tool names.
3. **Know your numbers.** Dev environment cost ~$160/month. RDS PITR window: 35 days. ArgoCD poll interval: 3 minutes. EKS node count: 3 × t3.small. These details signal real experience.
4. **Behavioral questions:** Use the STAR format (Situation → Task → Action → Result). Reference specific incidents from `07-behavioral-realtime.md`.
5. **When you don't know something:** Say "I haven't used that specific tool, but the concept maps to X which we use in our project — in X, the way to handle this is..." Never just say "I don't know."

---

## Source Repos

Questions consolidated from:
- [zen-infra](../zen-infra/) — Terraform, VPC, EKS, RDS, CI/CD scenarios
- [zen-pharma-backend](../zen-pharma-backend/) — DevSecOps pipeline, microservices CI
- [zen-gitops](../zen-gitops/) — ArgoCD, GitOps promotion, External Secrets, incident response
- [zen-pharma-frontend](../zen-pharma-frontend/) — React UI, Node.js build pipeline
