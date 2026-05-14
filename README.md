# gitops-cluster-hub

GitOps repository for bootstrapping the Exarep hub OpenShift cluster. Takes a fresh cluster to production-ready using Argo CD and Kustomize.

## Prerequisites

- Logged in to the hub cluster via `oc`
- AWS CLI configured with valid credentials
- Secrets stored in AWS Secrets Manager (via `iac/pb-setup-secrets.yaml`)

## Bootstrap

Run the bootstrap playbook to install the External Secrets Operator, create the AWS credentials secret on the cluster, and configure the ClusterSecretStore.

```shell
ansible-playbook pb-bootstrap-gitops.yaml
```

This playbook does the following:

1. Installs the External Secrets Operator (namespace, operator-group, subscription)
2. Waits for the operator to be ready
3. Creates the `aws-secrets-manager-credentials` secret in `openshift-external-secrets-operator`
4. Applies the `ClusterSecretStore` pointing to AWS Secrets Manager

Once the bootstrap is complete, apply the app-of-apps to hand control to Argo CD:

```shell
oc apply -f app-of-apps.yaml
```

## Repository Structure

```
gitops-cluster-hub/
├── app-of-apps.yaml                          # Root Application (project: cluster)
├── pb-bootstrap-gitops.yaml                  # Ansible bootstrap playbook
├── applications/                             # Argo CD Application manifests
├── openshift-external-secrets-operator/      # ESO operator + ClusterSecretStore
│   ├── kustomization.yaml
│   ├── namespace.yaml                        # sync-wave-0
│   ├── operator-group.yaml                   # sync-wave-1
│   ├── subscription.yaml                     # sync-wave-2
│   └── cluster-secret-store.yaml             # sync-wave-3
└── openshift-gitops-operator/                # GitOps operator + Argo CD instance
    ├── kustomization.yaml
    ├── namespaces.yaml                       # sync-wave-0
    ├── operator-group.yaml                   # sync-wave-1
    ├── subscription.yaml                     # sync-wave-2
    ├── exarep-gitops-cluster-argocd.yaml     # sync-wave-3
    ├── cluster-app-project.yaml              # sync-wave-4
    └── external-secret.yaml                  # sync-wave-5 (GitHub PAT for repo access)
```

## Secrets Flow

1. `iac/pb-setup-secrets.yaml` stores the GitHub PAT in AWS Secrets Manager as `exarep/github`
2. `pb-bootstrap-gitops.yaml` installs ESO and creates the `ClusterSecretStore`
3. `external-secret.yaml` pulls the PAT from Secrets Manager and creates an Argo CD repository secret
4. Argo CD uses the PAT to authenticate to `https://github.com/exarep/gitops-cluster-hub.git`
