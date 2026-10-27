# ref-gitops

The single source of truth for **what runs where**. Argo CD watches this repository
and makes the cluster match it. Nobody deploys with `kubectl` or `helm install`
outside quick experiments.

```
ref-gitops/
├── apps/
│   └── <app-name>/
│       ├── values-dev.yaml     image tag, replicas, resources for dev
│       └── values-prod.yaml    same for prod
└── argocd/
    ├── application-dev.yaml    auto-sync, prune, self-heal
    └── application-prod.yaml   manual sync
```

The chart itself lives in `ref-app-node/chart`. The Applications use two sources:
the chart from `ref-app-node`, the values from this repo.

## Before first use

Replace `YOUR_GITHUB_USER` (lowercase in the image path) in the four YAML files, and put
a real tag in `image.tag` (GitHub, Packages, `ref-app-node`, or the CI run summary).

## Validate locally

From a folder containing both `ref-app-node` and `ref-gitops`:

```bash
helm lint ./ref-app-node/chart -f ref-gitops/apps/crococoder-status/values-dev.yaml
helm template t ./ref-app-node/chart -f ref-gitops/apps/crococoder-status/values-dev.yaml
helm template t ./ref-app-node/chart -f ref-gitops/apps/crococoder-status/values-prod.yaml

# let the cluster check Pod Security and the quota, without creating anything
helm template t ./ref-app-node/chart -f ref-gitops/apps/crococoder-status/values-dev.yaml \
  | kubectl apply --dry-run=server -n dev -f -
```

## Deploy (Phase 5, one-time bootstrap)

Install Argo CD, register the repositories if they are private
(`argocd repo add ... --username ... --password <token>`, the token never goes in Git), then:

```bash
kubectl apply -f argocd/application-dev.yaml
kubectl apply -f argocd/application-prod.yaml
```

## Promotion flow

1. CI builds an image and updates `image.tag` in `values-dev.yaml`.
2. Argo CD syncs dev automatically.
3. When dev is verified, open a **merge request** copying the same tag into `values-prod.yaml`.
4. After the merge, press **Sync** on `crococoder-status-prod` in Argo CD.

## Adding another application

Create `apps/<new-app>/values-dev.yaml` and `values-prod.yaml`, and two Application files
copied from the ones here (change the name, the chart path and the values path).
Namespaces `dev` and `prod` already exist (Terraform), so nothing else is needed.

## Rules

- Image tags are exact: `sha-xxxxxxx` or `vX.Y.Z`. Never `latest`.
- No secrets in this repo, ever.
- Production changes only through a reviewed merge request.
