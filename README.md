# DevOps Lab GitOps

GitOps desired-state repository for the DevOps Platform Lab.

This repository defines the Kubernetes resources that should exist inside the platform's workload clusters. Infrastructure such as VPCs, IAM resources, ECR, and EKS clusters is managed separately in the `devops-lab-infra` repository.

## Repository Responsibilities

The project is separated into three primary repositories:

```text
devops-lab-app
    Application source, build configuration, and CI pipeline

devops-lab-infra
    Terraform infrastructure and Kubernetes cluster provisioning

devops-lab-gitops
    Kubernetes desired state and environment-specific configuration
```

This separation allows application code, infrastructure, and deployment state to evolve independently.

## Environment Architecture

The local platform currently contains three Kubernetes clusters:

```text
devops-mgmt
    Management and platform tooling

devops-nonprod
    ├── DEV
    └── UAT

devops-prod
    └── PROD
```

DEV and UAT share the non-production cluster while PROD runs in a separate cluster to provide a stronger isolation boundary.

The management cluster hosts Argo CD and is reserved for management and platform tooling.

## Repository Structure

```text
argocd/
├── root-app.yaml
└── applications/
    ├── kustomization.yaml
    ├── devops-nonprod.yaml
    └── devops-prod.yaml

clusters/
├── nonprod/
│   ├── kustomization.yaml
│   └── namespaces/
│       ├── dev.yaml
│       └── uat.yaml
│
└── prod/
    ├── kustomization.yaml
    └── namespaces/
        └── prod.yaml
```

Kustomize is used to compose the desired Kubernetes resources for each cluster and the declarative Argo CD Application definitions.

## GitOps Model

The intended deployment workflow is:

```text
Application source
       |
       v
     Jenkins
       |
       +--> Build
       +--> Test
       +--> Scan
       +--> Build container image
       +--> Push immutable image
       |
       v
Update GitOps desired state
       |
       v
     Argo CD
       |
       v
Kubernetes clusters
```

Jenkins is responsible for continuous integration.

Argo CD is responsible for continuous delivery and Kubernetes reconciliation.

Jenkins should not directly deploy workloads with `kubectl apply`.

## Argo CD Application Management

Argo CD runs in the `devops-mgmt` cluster and manages the two workload clusters:

```text
devops-mgmt
└── Argo CD
    ├── devops-nonprod
    │   ├── DEV
    │   └── UAT
    │
    └── devops-prod
        └── PROD
```

The Argo CD Applications are themselves declared in Git.

```text
argocd/root-app.yaml
        |
        v
devops-platform-apps
        |
        v
argocd/applications/
├── devops-nonprod.yaml
└── devops-prod.yaml
        |
        +--> clusters/nonprod
        |
        └--> clusters/prod
```

`devops-platform-apps` is the bootstrap root Application. After it is created in the management cluster, it reconciles the child Application definitions stored in this repository.

The child Applications refer to workload clusters by their logical Argo CD registration names:

```text
devops-nonprod
devops-prod
```

Runtime cluster credentials, bearer tokens, certificate authority data, and Floci-specific API endpoints are not stored in this repository. They remain in Argo CD cluster credential Secrets inside the management cluster.

Synchronization currently remains manual so reconciliation behavior can be inspected explicitly while the platform is being built.

## Artifact Promotion

The project follows a build-once promotion model.

An application container image should be built once and identified by an immutable image digest:

```text
DEV
 |
 v
UAT
 |
 v
PROD
```

The same artifact is promoted between environments rather than rebuilt separately for each environment.

## Current Desired State

### Non-production cluster

Cluster:

```text
devops-nonprod
```

Namespaces:

```text
dev
uat
```

### Production cluster

Cluster:

```text
devops-prod
```

Namespace:

```text
prod
```

## Validation

Render non-production configuration:

```bash
kubectl kustomize clusters/nonprod
```

Render production configuration:

```bash
kubectl kustomize clusters/prod
```

Validate the desired state against the Kubernetes API without persisting resources:

These server-side dry-run commands were used during Milestone 7 to verify the namespace manifests. The namespaces were intentionally not created manually because future deployment ownership belongs to Argo CD.

```bash
kubectl \
  --context devops-nonprod \
  apply \
  --dry-run=server \
  -k clusters/nonprod
```

```bash
kubectl \
  --context devops-prod \
  apply \
  --dry-run=server \
  -k clusters/prod
```

## Relevant Project Milestone

Milestone 7 GitOps foundation completed:

```text
Separate GitOps repository established
DEV namespace declared for devops-nonprod
UAT namespace declared for devops-nonprod
PROD namespace declared for devops-prod
Kustomize entry points created
Desired state validated with Kubernetes server-side dry-run
Namespaces intentionally not applied manually
```

The namespace manifests remain desired state in Git until Argo CD is introduced as the reconciliation layer.

## Current Status

Implemented:

* Repository structure
* DEV namespace definition
* UAT namespace definition
* PROD namespace definition
* Kustomize entry points for non-production and production
* Argo CD installed in the management cluster
* Non-production and production workload clusters registered with Argo CD
* GitOps repository connected to Argo CD
* `devops-nonprod` Application
* `devops-prod` Application
* Git-managed DEV, UAT, and PROD namespaces
* Manual synchronization workflow
* Drift detection and reconciliation verification
* Declarative child Application definitions
* Root Application definition for Application bootstrapping

Planned:

* Spring Boot Deployment and Service manifests
* Environment-specific application configuration
* Health probes
* Resource requests and limits
* Network policies
* Immutable image promotion
* Jenkins-driven GitOps updates
* Automated synchronization policy where appropriate

## Security

Secrets and generated Kubernetes credentials must not be committed to this repository.

Sensitive runtime configuration will be handled separately as the platform evolves.

