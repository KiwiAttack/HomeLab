# HomeLab Kubernetes GitOps Setup

Welcome to my Homelab GitOps repository! This project defines a self-hosted infrastructure stack using [Kubernetes](https://kubernetes.io/), [Flux](https://fluxcd.io/), and secure GitOps practices. The entire system runs on bare-metal nodes at home with persistent storage, service monitoring, and secret encryption.

---

## Core Technologies

- **Kubernetes (k3s)** – Lightweight Kubernetes distribution for edge computing
- **FluxCD** – GitOps controller to manage Kubernetes state from this repo
- **Kustomize** – Kubernetes-native configuration management
- **SOPS + Age** – Encrypted Kubernetes Secrets with Git-safe commits
- **Traefik / Ingress-NGINX** – Ingress controllers with TLS termination
- **cert-manager** – Automated certificate management using Let's Encrypt
- **Persistent Storage** – Backed by local NFS or ZFS with PVCs
- **CrowdSec** – Security automation and protection (under active deployment)

---

## Repository Structure

```text
.
├── apps/                   # Namespace-scoped applications
│   ├── vaultwarden/        # Encrypted password manager
│   ├── speedtest/          # Internet speed monitoring
│   └── wiki/               # Self-hosted Wiki + Postgres backend
├── infrastructure/         # Cluster-wide tools (monitoring, ingress, certs)
│   ├── blackbox/           # Blackbox Exporter monitoring probes
│   ├── crowdsec/           # CrowdSec LAPI and dashboards
│   └── cert-manager/       # Certificate automation
├── .gitlab-ci.yml          # YAML/manifest validation pipeline
├── README.md               # This file
└── LICENSE

```

## Secrets Management
All sensitive credentials are managed with [SOPS](https://github.com/getsops/sops) and [Age](https://github.com/FiloSottile/age). Encrypted files are committed to Git and decrypted at runtime in-cluster via Flux + KSOPS.

Example of encrypting a secret:

```bash
sops --encrypt --age <YOUR_AGE_PUBLIC_KEY> --in-place ./secrets/secret.yaml
Decryption is only possible in-cluster with the correct private Age key stored securely.
```

## Deployment Flow
Create the GitOps repository (this repo).

- Bootstrap Flux with your Git credentials and public Age key:

```bash
flux bootstrap github \
  --owner=<your-github-user> \
  --repository=HomeLab \
  --path=./ \
  --personal \
  --private=false
```

- Apply secrets and allow SOPS to decrypt them via KSOPS or controller configuration.
- Flux continuously syncs all resources from Git.

## Monitoring and Observability

- [Blackbox Exporter](https://github.com/prometheus/blackbox_exporter) to probe endpoints like external IPs, DNS, or self-hosted services.
- [Prometheus](https://github.com/prometheus/prometheus) to track metrics.
- Alerts and dashboards are set up using Alertmanager and [Grafana](https://github.com/grafana/grafana) for visual dashboards.

## Network Layout (High-Level)
```mermaid
graph TD
  GitHub[GitHub Repo] -->|Flux Sync| Cluster[k3s Cluster]
  Cluster -->|Ingress| NGINX[Ingress-NGINX]
  Cluster -->|Service| Vaultwarden
  Cluster -->|Service| Wiki
  Cluster -->|Service| Speedtest
  Cluster -->|PVC| NFS[NFS Storage]
```

## CI/CD

- This repository includes a GitLab CI pipeline to validate manifests:
- yamllint checks formatting
- kustomize build ensures Kustomizations are valid

```yaml
before_script:
  - curl -Lo /usr/local/bin/kustomize ...
script:
  - find . -name "*.yaml" | xargs yamllint
  - kustomize build clusters/heimdall > /dev/null
```