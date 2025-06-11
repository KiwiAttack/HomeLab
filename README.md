# HomeLab: GitOps with k3s, Flux, and a Secure Wiki

This repository contains my personal homelab setup, managed entirely through GitOps principles. The core of this setup is a Kubernetes cluster running k3s, with application and infrastructure deployments orchestrated by Flux CD. A key feature is a TLS-secured Wiki application, leveraging NGINX Ingress and Cert-Manager for automatic certificate provisioning from Let's Encrypt.

## Table of Contents

- [HomeLab: GitOps with k3s, Flux, and a Secure Wiki](#homelab-gitops-with-k3s-flux-and-a-secure-wiki)
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
    - [PostgreSQL](#postgresql)
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

* **NGINX Ingress Controller**: Manages external access to services within the cluster, providing HTTP/S routing and advanced traffic management features. It is configured with custom logging and integrated with Crowdsec for basic WAF capabilities.

### Certificate Management

* **Cert-Manager**: Automates the issuance and renewal of TLS certificates from various issuing sources, including Let's Encrypt.
    * `letsencrypt-prod`: ClusterIssuer for production-ready certificates from Let's Encrypt.
    * `letsencrypt-staging`: ClusterIssuer for testing certificates from Let's Encrypt staging environment.

### Persistent Storage

* **NFS (Network File System)**: Used for providing persistent storage to stateful applications like Wiki.js and PostgreSQL. The NFS server is located at `192.168.178.200`.

### Secret Management

* **Mozilla SOPS**: Secrets are encrypted using SOPS and stored in the Git repository. Flux's Kustomize controller is configured to decrypt these secrets before applying them to the cluster, ensuring sensitive information remains encrypted at rest and in transit.

## Applications

### Wiki.js

* A powerful and extensible Wiki application deployed on Kubernetes.
* **Database**: Uses PostgreSQL for its backend database.
* **Persistent Storage**: Data is stored on an NFS share for persistence.
* **Access**: Exposed via NGINX Ingress with automatic TLS from Let's Encrypt.

### PostgreSQL

* A robust, open-source relational database used as the backend for Wiki.js.
* **Persistent Storage**: Data is stored on an NFS share for persistence.

## Directory Structure
.
├── apps/
│   └── wiki/                   # Wiki.js application manifests
│       ├── base/
│       │   ├── deployment.yaml
│       │   ├── ingress.yaml
│       │   ├── namespace.yaml
│       │   └── service.yaml
│       ├── postgres/           # PostgreSQL manifests for Wiki.js
│       │   ├── deployment.yaml
│       │   ├── secret.sops.yaml
│       │   └── service.yaml
│       ├── storage/            # Persistent Volume/Claim definitions
│       │   ├── pv-postgres.yaml
│       │   └── pv-wiki.yaml
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
│   │   ├── cert-issuer.yaml
│   │   └── cert-staging-issuer.yaml
│   └── kustomization.yaml
├── cert-manager/           # Cert-Manager HelmRelease and repository
│   ├── base/
│   │   ├── helm-release.yaml
│   │   ├── jetstack-repo.yaml
│   │   └── namespace.yaml
│   └── kustomization.yaml
└── ingress-nginx/          # NGINX Ingress Controller
├── base/
│   ├── custom-nginx-config.yaml
│   ├── helm-release.yaml
│   ├── namespace.yaml
│   ├── nginx-repo.yaml
│   └── secret-api-key.yaml
└── kustomization.yaml

## Deployment

To deploy this homelab setup, you would typically:

1.  **Install k3s** on your bare-metal nodes.
2.  **Install Flux CD CLI** and bootstrap Flux onto your k3s cluster, pointing it to this Git repository.
    ```bash
    flux bootstrap github \
      --owner=<your-github-username> \
      --repository=HomeLab \
      --branch=main \
      --path=./clusters/heimdall \
      --personal=true \
      --token-auth \
      --read-write-key \
      --passphrase=<your-sops-age-key-passphrase>
    ```
    *Note: Replace `<your-github-username>` and `<your-sops-age-key-passphrase>` with your actual values.*

Flux will then automatically synchronize the cluster with the configurations defined in this repository, deploying all infrastructure components and applications.

## Configuration & Customization

* **Secrets**: All sensitive data (like database passwords and API keys) are managed using SOPS. To decrypt and encrypt secrets, ensure you have the correct SOPS age key configured.
* **NFS Paths**: Adjust the `path` and `server` values in `apps/wiki/storage/pv-*.yaml` to match your NFS server configuration.
* **Ingress Hostnames**: Update the `host` in `apps/wiki/base/ingress.yaml` to your desired domain for the Wiki.js application.
* **Cert-Manager Email**: Change the `email` in `infrastructure/cert-issuer/base/cert-issuer.yaml` and `cert-staging-issuer.yaml` to your email address for Let's Encrypt notifications.

## Contributing

Feel free to open issues or pull requests if you have suggestions or improvements for this homelab setup.

## License

This project is licensed under the MIT License - see the LICENSE file for details.