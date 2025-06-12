# HomeLab: GitOps with k3s, Flux, and Secure Services

This repository contains my personal homelab setup, managed entirely through GitOps principles. The core of this setup is a Kubernetes cluster running k3s, with application and infrastructure deployments orchestrated by Flux CD. Key services include a TLS-secured Wiki application and Vaultwarden (a Bitwarden compatible server), leveraging NGINX Ingress and Cert-Manager for automatic certificate provisioning from Let's Encrypt.

## Table of Contents

- [HomeLab: GitOps with k3s, Flux, and Secure Services](#homelab-gitops-with-k3s-flux-and-secure-services)
  - [Table of Contents](#table-of-contents)
  - [Overview](#overview)
  - [Cluster Components](#cluster-components)
    - [Kubernetes Distribution](#kubernetes-distribution)
    - [GitOps Engine](#gitops-engine)
    - [Ingress Controller](#ingress-controller)
    - [Certificate Management](#certificate-management)
    - [Persistent Storage](#persistent-storage)
    - [Secret Management](#secret-management)
  - [Applications](#applications)
    - [Wiki.js](#wikijs)
    - [PostgreSQL (for Wiki.js)](#postgresql-for-wikijs)
    - [Vaultwarden](#vaultwarden)
  - [Directory Structure](#directory-structure)
  - [Deployment](#deployment)
  - [Configuration & Customization](#configuration--customization)
  - [Contributing](#contributing)
  - [License](#license)

## Overview

This homelab aims to demonstrate and utilize modern cloud-native practices, specifically GitOps, for managing personal services. All configurations are stored in this Git repository, and Flux CD ensures that the state of the Kubernetes cluster converges to the desired state defined here.

## Cluster Components

### Kubernetes Distribution

* **k3s**: A lightweight, certified Kubernetes distribution ideal for edge, IoT, and homelab environments. It's designed for low resource consumption while providing a complete Kubernetes experience.

### GitOps Engine

* **Flux CD**: Flux is a set of GitOps tools for keeping Kubernetes clusters in sync with configuration sources (like Git repositories) and automating updates to configuration when there is new code.
    * **Source Controller**: Manages fetching content from various sources (Git, Helm repositories, S3 buckets).
    * **Kustomize Controller**: Reconciles Kustomization resources, applying templated Kubernetes manifests to the cluster.
    * **Helm Controller**: Manages Helm chart releases.
    * **Notification Controller**: Handles events from Flux components and dispatches them to configured providers (e.g., webhooks for external alerting).

### Ingress Controller

* **NGINX Ingress Controller**: Manages external access to services within the cluster, providing HTTP/S routing and advanced traffic management features. It is configured with custom logging and integrated with Crowdsec for basic WAF capabilities using a Lua bouncer plugin.

### Certificate Management

* **Cert-Manager**: Automates the issuance and renewal of TLS certificates from various issuing sources, including Let's Encrypt.
    * `letsencrypt-prod`: ClusterIssuer for production-ready certificates from Let's Encrypt.
    * `letsencrypt-staging`: ClusterIssuer for testing certificates from Let's Encrypt staging environment.

### Persistent Storage

* **NFS (Network File System)**: Used for providing persistent storage to stateful applications like Wiki.js and Vaultwarden. The NFS server is located at `192.168.178.200`.

### Secret Management

* **Mozilla SOPS**: Sensitive data (like database passwords and API keys) are encrypted using SOPS and stored in the Git repository. Flux's Kustomize controller is configured to decrypt these secrets before applying them to the cluster, ensuring sensitive information remains encrypted at rest and in transit.

## Applications

### Wiki.js

* A powerful and extensible Wiki application deployed on Kubernetes.
* **Database**: Uses PostgreSQL for its backend database.
* **Persistent Storage**: Data is stored on an NFS share for persistence.
* **Access**: Exposed via NGINX Ingress with automatic TLS from Let's Encrypt.

### PostgreSQL (for Wiki.js)

* A robust, open-source relational database used as the backend for Wiki.js.
* **Persistent Storage**: Data is stored on an NFS share for persistence.

### Vaultwarden

* A light-weight Bitwarden® compatible server, perfect for self-hosting your password manager.
* **Persistent Storage**: Uses NFS at `/srv/k8s-data/vaultwarden` for data persistence.
* **Email Notifications**: Configured to send emails via Gmail SMTP on port 465 for features like account verification, using an App Password for secure authentication.
* **Security**: Includes rate-limiting to protect against brute-force attacks and is secured with an Argon2-hashed admin token.
* **Access**: Exposed via NGINX Ingress with automatic TLS from Let's Encrypt on `https://vault.kiwilab.dev`.

## Directory Structure
```bash
.
├── apps/
│   ├── wiki/                   # Wiki.js application manifests
│   │   ├── base/
│   │   ├── postgres/
│   │   ├── storage/
│   │   └── kustomization.yaml
│   └── vaultwarden/            # Vaultwarden application manifests
│       ├── base/
│       │   ├── deployment.yaml
│       │   ├── ingress.yaml
│       │   ├── namespace.yaml
│       │   └── service.yaml
│       ├── storage/
│       │   └── pv-vaultwarden.yaml
│       ├── secret.sops.yaml    # Encrypted admin token
│       └── kustomization.yaml
├── clusters/
│   └── heimdall/               # Cluster-specific configurations
│       ├── flux-system/        # Flux CD core components and synchronization
│       │   ├── gotk-components.yaml
│       │   ├── gotk-sync.yaml
│       │   └── kustomization.yaml
│       ├── apps.yaml           # Kustomization for applications
│       └── infrastructure.yaml # Kustomization for infrastructure components
└── infrastructure/
├── cert-issuer/            # Cert-Manager ClusterIssuers
│   ├── base/
│   └── kustomization.yaml
├── cert-manager/           # Cert-Manager HelmRelease and repository
│   ├── base/
│   └── kustomization.yaml
└── ingress-nginx/          # NGINX Ingress Controller
├── base/
└── kustomization.yaml
```

## Deployment

To deploy this homelab setup, you would typically:

1.  **Install k3s** on your bare-metal nodes.
2.  **Install Flux CD CLI** and bootstrap Flux onto your k3s cluster, pointing it to this Git repository.
    ```bash
    flux bootstrap github \
      --owner=<your-github-username> \
      --repository=<your-repo-name> \
      --branch=main \
      --path=./clusters/<your-cluster-name> \
      --personal \
    ```
    *Note: Replace `<your-github-username>` and `<your-sops-age-key-passphrase>` with your actual values.*

Flux will then automatically synchronize the cluster with the configurations defined in this repository, deploying all infrastructure components and applications.

## Configuration & Customization

* **Secrets**: All sensitive data (like database passwords and API keys) are managed using SOPS. To decrypt and encrypt secrets, ensure you have the correct SOPS age key configured.
* **NFS Paths**: Adjust the `path` and `server` values in `apps/*/storage/pv-*.yaml` to match your NFS server configuration.
* **Ingress Hostnames**: Update the `host` in `apps/*/base/ingress.yaml` files to your desired domains for Wiki.js, Vaultwarden, etc.
* **Cert-Manager Email**: Change the `email` in `infrastructure/cert-issuer/base/cert-issuer.yaml` and `cert-staging-issuer.yaml` to your email address for Let's Encrypt notifications.
* **Vaultwarden Specifics**:
    * **Admin Token**: Generate a secure Argon2 hash for your admin token using `vaultwarden hash <your-token>` inside the Vaultwarden container and update `apps/vaultwarden/secret.sops.yaml`.
    * **SMTP Settings**: Configure `SMTP_HOST`, `SMTP_PORT`, `SMTP_USERNAME`, `SMTP_PASSWORD` (via secret), `SMTP_SSL`, `SMTP_STARTTLS`, `SMTP_EXPLICIT_TLS` for email functionality.
    * **Domain**: Set `DOMAIN` and `URL_BASE` to your public URL (e.g., `https://vault.kiwilab.dev`) for correct link generation.
    * **Sign-ups**: Adjust `SIGNUPS_ALLOWED` (`true` to enable, `false` to disable) and `SIGNUPS_VERIFY` (`true` to require email verification, `false` to skip) as needed for user registration. Remember to disable sign-ups after creating initial accounts for security.

## Contributing

Feel free to open issues or pull requests if you have suggestions or improvements for this homelab setup.

## License

This project is licensed under the MIT License - see the LICENSE file for details.