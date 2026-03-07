# OpenShift Deployment with Argo CD + Kustomize

This directory provides a GitOps-friendly deployment path for Buttercup on OpenShift using:

- **Kustomize** to compose deployment resources
- **Helm** (via Kustomize `helmCharts`) to render the Buttercup chart
- **Argo CD / OpenShift GitOps** to continuously reconcile the deployment

## Files in this directory

- `kustomization.yaml`: Kustomize entrypoint that renders the local Helm chart in `deployment/k8s`
- `values-openshift.yaml`: OpenShift-oriented values overrides
- `namespace.yaml`: creates the `buttercup` namespace
- `argocd-application.yaml`: example Argo CD `Application`

## Prerequisites

1. OpenShift cluster admin access (or equivalent delegated permissions)
2. OpenShift GitOps installed (`openshift-gitops` namespace)
3. RWX/RWO storage classes available (example uses OpenShift Data Foundation classes)
4. Container pull credentials for GHCR (`ghcr-auth` and `docker-auth` secrets)

## 1) Configure values

Edit `values-openshift.yaml` and replace placeholders:

- `global.crs.hostname`
- LLM credentials under `litellm`
- storage classes under `volumes.*.storageClass` and `redis.master.persistence.storageClass`

> Recommended: keep secrets out of Git. In production, use Sealed Secrets, External Secrets Operator, or Argo CD Vault Plugin.

## 2) Create required pull secrets

Create pull secrets in the target namespace (`buttercup`):

```bash
oc -n buttercup create secret docker-registry ghcr-auth \
  --docker-server=ghcr.io \
  --docker-username='<github-username>' \
  --docker-password='<github-pat>'

oc -n buttercup create secret docker-registry docker-auth \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username='<dockerhub-username>' \
  --docker-password='<dockerhub-token>'
```

## 3) (Optional) validate manifests locally

From the repository root:

```bash
kustomize build deployment/k8s/openshift --enable-helm
```

## 4) Deploy with Argo CD

1. Update `argocd-application.yaml` with your `repoURL` and `targetRevision`.
2. Apply the Argo CD application:

```bash
oc apply -f deployment/k8s/openshift/argocd-application.yaml
```

3. Verify sync and workload status:

```bash
oc -n openshift-gitops get applications.argoproj.io buttercup
oc -n buttercup get pods
```

## OpenShift-specific notes

- `global.volumes.nodeLocal.enabled` is set to `false` because hostPath-based node-local storage is typically restricted.
- `dind-daemon` may require a privileged SCC depending on your cluster policy. If your platform team disallows privileged pods, coordinate an alternative build/runtime strategy before deployment.
