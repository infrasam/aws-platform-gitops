# AWS Platform GitOps

![ArgoCD](https://img.shields.io/badge/ArgoCD-EF7B4D?style=for-the-badge&logo=argo&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)
![Helm](https://img.shields.io/badge/Helm-0F1689?style=for-the-badge&logo=helm&logoColor=white)
![cert-manager](https://img.shields.io/badge/cert--manager-326CE5?style=for-the-badge&logoColor=white)

The GitOps side of a Kubernetes platform on AWS. **Argo CD** installs everything on the cluster from this repository, using a single **ApplicationSet** that combines cluster metadata with the components listed per environment.

The cluster, Argo CD and the connection to this repository are provisioned by Terraform in [aws-platform-iac](https://github.com/infrasam/aws-platform-iac).

---

## Architecture Overview

```mermaid
flowchart LR
    subgraph iac["📦 aws-platform-iac"]
        tf["🛠 Terraform"]
    end

    subgraph cluster["☸ Kubernetes Cluster"]
        secret["🔑 Cluster Secret<br/><code>environment=dev</code>"]
        boot["🚀 bootstrap Application"]
        appset["ApplicationSet<br/><code>infrastructure</code>"]
        cert["🔐 cert-manager"]
    end

    subgraph repo["📦 This Repository"]
        bootstrap["bootstrap/"]
        envs["environments/dev/infrastructure/"]
        values["infrastructure/"]
    end

    tf ==>|"Creates"| secret
    tf ==>|"Creates"| boot
    boot -->|"Syncs"| bootstrap
    bootstrap --> appset
    secret -->|"Selects environment"| appset
    envs -->|"Components and versions"| appset
    appset --> cert
    values -.->|"Helm values"| cert

    style iac fill:#161b22,stroke:#7b42bc,color:#9a6dd7,stroke-width:2px
    style cluster fill:#161b22,stroke:#3fb950,color:#3fb950,stroke-width:2px
    style repo fill:#161b22,stroke:#ef7b4d,color:#ef7b4d,stroke-width:2px
```

**How it works:**

1. Terraform registers the cluster in Argo CD with a **cluster Secret** labelled with its environment, and creates one **bootstrap** Application
2. The bootstrap Application syncs `bootstrap/`, which contains the `infrastructure` ApplicationSet
3. The ApplicationSet uses a **matrix generator**: for every registered cluster, it reads every file in `environments/<environment>/infrastructure/`
4. Each file becomes one Application that installs a Helm chart with the version from that file, the shared values from `infrastructure/<name>/` and, if present, the environment's overrides from `environments/<environment>/infrastructure/values/`

---

## Components

| Component | Description |
|-----------|-------------|
| **cert-manager** | Issues and renews TLS certificates, CRDs installed with the chart |

---

## Repository Structure

```
.
├── bootstrap/                      # Synced by the bootstrap Application
│   └── applicationset-infra.yaml   #   ApplicationSet for all infrastructure components
│
├── environments/                   # WHAT runs where, and which version
│   └── dev/
│       └── infrastructure/
│           ├── cert-manager.yaml   #   Chart, repository, version, namespace
│           └── values/             #   Optional overrides for this environment
│
├── infrastructure/                 # HOW a component is configured
│   └── cert-manager/
│       └── values.yaml             #   Helm values shared by all environments
│
├── .github/workflows/ci.yaml       # yamllint, render and validate, gitleaks
├── .pre-commit-config.yaml         # gitleaks and yamllint before every commit
├── .yamllint.yaml                  # YAML rules for the repository
└── renovate.json                   # Automated chart and tool updates
```

---

## Validation

Every pull request renders each component exactly as Argo CD would, before anything reaches a cluster.

```mermaid
flowchart LR
    pr["📬 Pull Request"] --> lint["🧹 yamllint"]
    pr --> render["⚙ helm template<br/><code>per environment and component</code>"]
    render --> kc["✅ kubeconform<br/><code>Kubernetes and CRD schemas</code>"]
    pr --> leaks["🔐 gitleaks"]
    lint --> main["🔒 main"]
    kc --> main
    leaks --> main

    style render fill:#1a2332,stroke:#58a6ff,color:#adbac7
    style kc fill:#1f2a1f,stroke:#3fb950,color:#adbac7
    style main fill:#161b22,stroke:#ef7b4d,color:#ef7b4d,stroke-width:2px
```

- `helm template` uses the chart version from the environment file and the same value files as the ApplicationSet, shared values first and environment overrides last
- `kubeconform` validates the output against the Kubernetes schemas and, for custom resources, the CRDs catalog
- `main` only accepts pull requests with all checks green

---

## Dependency Updates

**Renovate** watches every chart in `environments/dev/infrastructure/` and opens a pull request when a new version is released. The pull request goes through the same validation as any other change, so a new chart version is rendered and checked before Argo CD ever sees it.

```mermaid
sequenceDiagram
    participant R as Renovate
    participant GH as GitHub
    participant CI as CI
    participant A as Argo CD

    R->>GH: Detects a new chart version
    R->>GH: Opens PR bumping environments/dev
    CI->>GH: Renders and validates the new version
    actor Dev as Developer
    Dev->>GH: Reviews and merges
    A->>GH: Detects the change
    A->>A: Upgrades the component in dev
```

Only dev is updated automatically. Other environments receive a version once it has run in dev. Tool versions in the workflows (Helm, kubeconform) and GitHub Actions are kept up to date the same way.

---

## Adding a Component

1. Create `environments/<environment>/infrastructure/<name>.yaml` with the chart, repository, version and namespace
2. Create `infrastructure/<name>/values.yaml` with its settings
3. Open a pull request. After merge, the ApplicationSet discovers the file and Argo CD installs the component

No change to the ApplicationSet is needed.

---

## Key Design Decisions

- **ApplicationSet instead of app of apps**: one template creates every Application, so adding a component never means writing another Application manifest
- **Versions per environment, values layered**: each environment pins its own chart versions, and shared values can be overridden per environment in `environments/<environment>/infrastructure/values/`, so a change is tested in dev before it reaches other environments
- **Environment comes from the cluster**: the cluster Secret decides which folder a cluster reads, so a new cluster with the same label gets the same components automatically
- **Multi-source Applications**: the chart comes from its upstream Helm repository and the values from this repository, so no charts are vendored
- **One entry point**: Terraform creates a single object and knows nothing about the content of this repository
