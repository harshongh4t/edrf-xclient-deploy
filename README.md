# Helm Chart — eDRF XClient Container Security

## What is Helm?

Helm is a package manager for Kubernetes/OpenShift. Instead of applying multiple YAML files one by one using `oc apply -f`, Helm packages them into a **chart** and deploys everything with a single command.


## Chart Structure

```
edrf-xclient/
├── Chart.yaml          # Chart metadata (name, version, description)
├── values.yaml         # All configurable values (EPO IP, image, resources)
└── templates/
    ├── 01-serviceaccount.yaml   # ServiceAccount + RoleBinding for privileged SCC
    ├── 02-epo-details.yaml      # ConfigMap + Secret for EPO credentials
    └── 03-deploy-edrf.yaml        # DaemonSet that runs XClient on every node
```

---

## Chart.yaml

Identity card of the Helm chart. Required by Helm to identify and version the chart.


## values.yaml

Single place for all configurable values. No need to edit individual YAML templates. Alongside
the ePO connection, image, and resource settings, the `deploymentDetails` block
(`clusterName`, `deploymentMode`) sets the deployment-identity metadata reported by the agent.

The `epo` (`ip`, `user`, `password`), `image` (`repository`, `digest`), and
`deploymentDetails` (`clusterName`, `deploymentMode`) values are **required** and have no
usable defaults — they are enforced by Helm's `required` function, so `helm install`/`upgrade`
fails fast if any are left empty. `epo.password` is required only when `epo.existingSecret` is
not set.

## DXL & Deployment Identity

- The container image bundles **DXL** (Data Exchange Layer), installed from the TA package
  during the image build (see `rhel10/Dockerfile`).
- The DaemonSet (`03-deploy-edrf.yaml`) injects identity env vars for Kubernetes telemetry:
  `CLUSTER_NAME` / `DEPLOYMENT_MODE` (from `deploymentDetails`), `POD_NAME` (downward API), and
  `TRELLIX_KUBERNETES_ENVIRONMENT: "1"`. It also mounts the host's `/etc/os-release` at
  `/host-etc-os-release` so the agent reports the host OS release, not the container's.

## Commands

### Pre-Install Validation

```bash
# Check chart for YAML syntax errors, missing fields, invalid templates
helm lint <chart_path>

# Preview exactly what will be deployed without actually deploying
helm upgrade --install edrf-xclient <chart_path> \
  -n <namespace> --dry-run
```

---

### Install

```bash
# Standard install — creates namespace if it doesn't exist
helm upgrade --install edrf-xclient <chart_path> \
  -n <namespace> --create-namespace

# Install with password passed at runtime (recommended — never store password in values.yaml)
helm upgrade --install edrf-xclient <chart_path> \
  -n <namespace> \
  --set epo.password="<epo_password>"


oc create secret generic <secret_name> \
  --from-literal=epo-passwd=<epo_password> \
  -n <namespace>
```

---

### Artifactory Pull Secret (Run Once Before Install)

Required for OpenShift to pull the container image from Artifactory private registry.

```bash
oc create secret generic <image_pull_secret> \
  --from-file=.dockerconfigjson=config.json \
  --type=kubernetes.io/dockerconfigjson \
  -n <namespace>

oc secrets link <service_account_name> <image_pull_secret> --for=pull -n <namespace>
```

---

### EPO Secret (Alternative to passing via --set)

Admin creates the secret once manually. Helm never needs to know the password.

```bash
oc create secret generic <secret_name> \
  --from-literal=epo-passwd="<epo_password>" \
  -n <namespace>
```

---

### Inspect Deployed Release

```bash
# List all helm releases across all namespaces
helm list --all-namespaces

# List releases in specific namespace
helm list -n <namespace>

# Detailed status of a release
helm status edrf-xclient -n <namespace>

# See current values in use
helm get values edrf-xclient -n <namespace> --all

# See rendered templates that were applied to cluster
helm template edrf-xclient <chart_path> -n <namespace>
```

---

### Upgrade

```bash
# Upgrade after changes to values.yaml or templates
helm upgrade edrf-xclient <chart_path> -n <namespace>
```

---

### History and Rollback

```bash
# View full deployment history with revision numbers
helm history edrf-xclient -n <namespace>

# Rollback to a specific revision (e.g. revision 1)
helm rollback edrf-xclient 1 -n <namespace>
```

---

### Uninstall

```bash
# Remove everything deployed by Helm in one command
helm uninstall edrf-xclient -n <namespace>
```

Helm uninstall automatically deletes all resources it created:
- DaemonSet
- ServiceAccount
- RoleBinding
- ConfigMap
- Secret

> **Note:** The namespace (`edrf-tests`) must be deleted separately by the cluster admin:
> ```bash
> oc delete namespace edrf-tests