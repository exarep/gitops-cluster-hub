# gitops-cluster-hub

GitOps repository for managing the exarep hub cluster. Uses OpenShift GitOps (ArgoCD) with Kustomize overlays and sync waves.

## Prerequisites

- Access to an OpenShift cluster with `cluster-admin` privileges
- `oc` CLI authenticated to the target cluster
- Python 3.11+

## Setup

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
pip install --upgrade pip
ansible-galaxy collection install -r galaxy.yaml
```

## Pre-Bootstrap: Create AWS Credentials Secret

The External Secrets Operator requires AWS credentials to access Secrets Manager. Create the secret in the `openshift-config` namespace before bootstrapping:

```bash
oc create namespace openshift-config 2>/dev/null || true

oc create secret generic aws-credentials \
  -n openshift-config \
  --from-literal=access-key-id="$(aws configure get aws_access_key_id)" \
  --from-literal=secret-access-key="$(aws configure get aws_secret_access_key)"
```

## Bootstrap

```bash
source .venv/bin/activate
ansible-playbook pb-bootstrap.yaml
```

## Repository Structure

```
gitops-cluster-hub/
├── resources/
│   ├── openshift-gitops-operator/
│   │   ├── kustomization.yaml
│   │   ├── namespace.yaml
│   │   ├── operator-group.yaml
│   │   └── subscription.yaml
│   ├── gitops-cluster/
│   │   ├── kustomization.yaml
│   │   ├── namespace.yaml
│   │   └── argocd.yaml
│   ├── openshift-nmstate/
│   │   ├── kustomization.yaml
│   │   ├── namespace.yaml
│   │   ├── nmstate.yaml
│   │   ├── operator-group.yaml
│   │   └── subscription.yaml
│   ├── console-notification/
│   │   ├── kustomization.yaml
│   │   └── console-notification.yaml
│   ├── open-cluster-management/
│   │   ├── kustomization.yaml
│   │   ├── namespace.yaml
│   │   ├── operator-group.yaml
│   │   ├── subscription.yaml
│   │   ├── multiclusterhub.yaml
│   │   └── search.yaml
│   ├── openshift-pipelines/
│   │   ├── kustomization.yaml
│   │   ├── namespace.yaml
│   │   └── subscription.yaml
│   ├── console-plugins/
│   │   ├── kustomization.yaml
│   │   └── console.yaml
│   └── external-secrets/
│       ├── kustomization.yaml
│       ├── namespace.yaml
│       ├── operator-group.yaml
│       ├── subscription.yaml
│       └── cluster-secret-store.yaml
├── applications/
│   ├── kustomization.yaml
│   ├── openshift-gitops-operator.yaml
│   ├── gitops-cluster.yaml
│   ├── openshift-nmstate.yaml
│   ├── console-notification.yaml
│   ├── open-cluster-management.yaml
│   ├── openshift-pipelines.yaml
│   ├── console-plugins.yaml
│   ├── external-secrets.yaml
│   └── app-of-apps.yaml         # Root Application
├── pb-bootstrap.yaml
├── requirements.txt
└── README.md
```

## Details

### Bootstrap Playbook

The bootstrap playbook installs the OpenShift GitOps operator with the default ArgoCD instance disabled, then creates a custom ArgoCD instance and grants it cluster-admin privileges.

| Step | Description                                                          |
| ---- | -------------------------------------------------------------------- |
| 1    | Creates the`openshift-gitops-operator` namespace                   |
| 2    | Creates the OperatorGroup for the GitOps operator                    |
| 3    | Creates the Subscription with default ArgoCD instance disabled       |
| 4    | Waits for the ClusterServiceVersion to reach`Succeeded` phase      |
| 5    | Creates the`gitops-cluster` namespace                              |
| 6    | Creates the custom ArgoCD instance                                   |
| 7    | Waits for the ArgoCD instance to become`Available`                 |
| 8    | Enables the GitOps console plugin on the OpenShift web console       |
| 9    | Grants the ArgoCD application controller`cluster-admin` privileges |
| 10   | Applies the app-of-apps to hand control to ArgoCD                    |

The retry timeouts default to 10 minutes (60 retries x 10 seconds). Override them with extra vars if needed:

```bash
ansible-playbook pb-bootstrap.yaml \
  -e csv_retry_count=90 \
  -e csv_retry_delay=15
```

### App-of-Apps

The `applications/app-of-apps.yaml` is the root ArgoCD Application that points to the `applications/` directory. Each Application in that directory manages a corresponding `resources/` path with automated sync, self-heal, and pruning enabled.

| Sync Wave | Application               | Resource Path                       |
| --------- | ------------------------- | ----------------------------------- |
| 0         | openshift-gitops-operator | resources/openshift-gitops-operator |
| 1         | gitops-cluster            | resources/gitops-cluster            |
| 2         | openshift-nmstate         | resources/openshift-nmstate         |
| 3         | console-notification      | resources/console-notification      |
| 4         | open-cluster-management   | resources/open-cluster-management   |
| 5         | openshift-pipelines       | resources/openshift-pipelines       |
| 6         | console-plugins           | resources/console-plugins           |
| 7         | external-secrets          | resources/external-secrets          |

### Resource Manifests

The `resources/` directory contains the Kubernetes manifests managed by ArgoCD through the app-of-apps pattern.

**openshift-gitops-operator** — Operator installation via OLM:

| Sync Wave | Resource      |
| --------- | ------------- |
| 0         | Namespace     |
| 1         | OperatorGroup |
| 2         | Subscription  |

**gitops-cluster** — Custom ArgoCD instance:

| Sync Wave | Resource  |
| --------- | --------- |
| 0         | Namespace |
| 1         | ArgoCD    |

**openshift-nmstate** — Kubernetes NMState Operator:

| Sync Wave | Resource      |
| --------- | ------------- |
| 0         | Namespace     |
| 1         | OperatorGroup |
| 2         | Subscription  |
| 3         | NMState CR    |

**console-notification** — Cluster banner:

| Sync Wave | Resource            |
| --------- | ------------------- |
| 0         | ConsoleNotification |

**open-cluster-management** — Red Hat Advanced Cluster Management:

| Sync Wave | Resource        |
| --------- | --------------- |
| 0         | Namespace       |
| 1         | OperatorGroup   |
| 2         | Subscription    |
| 3         | MultiClusterHub |
| 4         | Search CR       |

**openshift-pipelines** — OpenShift Pipelines (Tekton):

| Sync Wave | Resource     |
| --------- | ------------ |
| 0         | Namespace    |
| 1         | Subscription |

**console-plugins** — Console operator plugin configuration:

| Sync Wave | Resource |
| --------- | -------- |
| 0         | Console  |

**external-secrets** — External Secrets Operator with AWS Secrets Manager:

| Sync Wave | Resource           |
| --------- | ------------------ |
| 0         | Namespace          |
| 1         | OperatorGroup      |
| 2         | Subscription       |
| 3         | ClusterSecretStore |
