# OpenShift Deployment with Argo CD + Kustomize

This directory provides GitOps-friendly deployment paths for Buttercup on OpenShift using:

- **Kustomize** to compose deployment resources
- **Helm** (via Kustomize `helmCharts`) to render the Buttercup chart
- **Argo CD / OpenShift GitOps** to continuously reconcile the deployment

## Available overlays

- `deployment/k8s/openshift`: standard OpenShift deployment
- `deployment/k8s/openshift/airgap`: OpenShift deployment for disconnected/air-gapped environments

## Standard OpenShift files

- `kustomization.yaml`: Kustomize entrypoint that renders the local Helm chart in `deployment/k8s`
- `values-openshift.yaml`: OpenShift-oriented values overrides
- `namespace.yaml`: creates the `buttercup` namespace
- `argocd-application.yaml`: example Argo CD `Application`

## Air-gapped OpenShift files

- `airgap/kustomization.yaml`: Kustomize entrypoint for disconnected clusters
- `airgap/values-openshift-airgap.yaml`: values override with internal registry images and safer pull policy defaults
- `airgap/argocd-application-airgap.yaml`: example Argo CD `Application` for the air-gapped overlay

## Prerequisites

1. OpenShift cluster admin access (or equivalent delegated permissions)
2. OpenShift GitOps installed (`openshift-gitops` namespace)
3. RWX/RWO storage classes available (example uses OpenShift Data Foundation classes)
4. Internal image registry reachable from all worker nodes
5. Mirrored images available in the internal registry

## 1) Configure values

### Standard OpenShift

Edit `values-openshift.yaml` and replace placeholders:

- `global.crs.hostname`
- LLM credentials under `litellm`
- storage classes under `volumes.*.storageClass` and `redis.master.persistence.storageClass`

### Air-gapped OpenShift

Edit `airgap/values-openshift-airgap.yaml` and replace all `REPLACE_ME` values.

At minimum, set:

- Internal image repositories and tags under `global.*Image`
- `global.ossFuzzContainerOrg` to an internal registry path
- CRS credentials under `global.crs.*`
- LLM credentials under `litellm.*`
- storage classes under `volumes.*.storageClass` and `redis.master.persistence.storageClass`

## 2) Required environment variables for air-gapped deployment

Set these variables before creating secrets / generating values in your CI pipeline:

| Variable | Required | Description |
|---|---|---|
| `AIRGAP_REGISTRY` | Yes | Internal registry host (example: `registry.apps.cluster.local`) |
| `AIRGAP_REGISTRY_USERNAME` | Yes | Username for internal registry pull secret |
| `AIRGAP_REGISTRY_PASSWORD` | Yes | Password/token for internal registry pull secret |
| `CRS_KEY_ID` | Yes | CRS API key ID |
| `CRS_KEY_TOKEN` | Yes | CRS API token |
| `CRS_KEY_TOKEN_HASH` | Yes | Argon2 hash of `CRS_KEY_TOKEN` |
| `COMPETITION_API_KEY_ID` | Yes | Competition API username/key ID |
| `COMPETITION_API_KEY_TOKEN` | Yes | Competition API password/token |
| `LITELLM_MASTER_KEY` | Yes | LiteLLM master key |
| `AZURE_API_BASE` | Optional* | Azure OpenAI base URL (required if using Azure model routes) |
| `AZURE_API_KEY` | Optional* | Azure OpenAI API key |
| `OPENAI_API_KEY` | Optional* | OpenAI API key (if used) |
| `ANTHROPIC_API_KEY` | Optional* | Anthropic API key (if used) |
| `BUTTERCUP_IMAGE_TAG` | Yes | Tag for mirrored Buttercup images |

## 3) Create required pull secret (air-gapped)

Create a pull secret in the target namespace (`buttercup`):

```bash
oc -n buttercup create secret docker-registry airgap-registry-auth \
  --docker-server="${AIRGAP_REGISTRY}" \
  --docker-username="${AIRGAP_REGISTRY_USERNAME}" \
  --docker-password="${AIRGAP_REGISTRY_PASSWORD}"
```

## 4) (Optional) Validate manifests locally

From the repository root:

```bash
kustomize build deployment/k8s/openshift --enable-helm
kustomize build deployment/k8s/openshift/airgap --enable-helm
```

## 5) Deploy with Argo CD

### Standard OpenShift

1. Update `argocd-application.yaml` with your `repoURL` and `targetRevision`.
2. Apply the Argo CD application:

```bash
oc apply -f deployment/k8s/openshift/argocd-application.yaml
```

### Air-gapped OpenShift

1. Update `airgap/argocd-application-airgap.yaml` with your `repoURL` and `targetRevision`.
2. Apply the Argo CD application:

```bash
oc apply -f deployment/k8s/openshift/airgap/argocd-application-airgap.yaml
```

## 6) Verify sync and workload status

```bash
oc -n openshift-gitops get applications.argoproj.io
oc -n buttercup get pods
```

## OpenShift-specific notes

- `global.volumes.nodeLocal.enabled` is set to `false` because hostPath-based node-local storage is typically restricted.
- `dind-daemon` may require a privileged SCC depending on your cluster policy. If your platform team disallows privileged pods, coordinate an alternative build/runtime strategy before deployment.
- In fully disconnected clusters, all third-party images (including dependencies like PostgreSQL/Redis) must be mirrored to an approved internal registry.
