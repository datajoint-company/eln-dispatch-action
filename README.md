# ELN Dispatch Action

Composite GitHub Action that triggers an ELN image tag update on [dj-gitops](https://github.com/datajoint-company/dj-gitops). After a dashboard image is built and pushed, this action dispatches a `repository_dispatch` event that creates an auto-PR to update the tenant's kustomize overlay `newTag`. Merging that PR triggers an ArgoCD sync to deploy the new image.

## Prerequisites

A GitHub PAT (or fine-grained token) with `repo` scope on `datajoint-company/dj-gitops`, stored as a repository or organization secret (e.g., `GITOPS_DISPATCH_TOKEN`).

## Usage

Add this step to your dashboard build workflow after the image push:

```yaml
- name: Update GitOps image tag
  uses: datajoint-company/eln-dispatch-action@v1
  with:
    tenant_name: org_project
    image_tag: ${{ steps.meta.outputs.version }}
    environment: prod
    token: ${{ secrets.GITOPS_DISPATCH_TOKEN }}
```

### Full workflow example

```yaml
name: Build and Deploy ELN Dashboard
on:
  push:
    tags:
      - '*'

jobs:
  build-and-dispatch:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build and push image
        # ... your existing build steps ...

      - name: Update GitOps image tag
        uses: datajoint-company/eln-dispatch-action@v1
        with:
          tenant_name: org_project
          image_tag: ${{ github.ref_name }}
          environment: prod
          token: ${{ secrets.GITOPS_DISPATCH_TOKEN }}
```

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `tenant_name` | Yes | | ELN tenant overlay directory name (e.g., `org_project`). Must match `[a-zA-Z0-9_-]+`. |
| `image_tag` | Yes | | Image tag to deploy (e.g., `1.2.0-sha-abc123`). Must match `[a-zA-Z0-9._-]+`. |
| `environment` | Yes | `prod` | Target environment: `qa` or `prod`. |
| `token` | Yes | | GitHub token with `repo` scope on `datajoint-company/dj-gitops`. |

## What happens after dispatch

1. `dj-gitops` receives the `eln-image-update` event
2. `update_service_images.py` validates the payload and updates `newTag` in `applications/k8s/kustomization/eln/overlay/{environment}/{tenant_name}/kustomization.yaml`
3. `peter-evans/create-pull-request` creates a PR titled `chore: update {tenant_name} ({environment})`
4. Merging the PR triggers ArgoCD to sync the new image (~3-5 min)
