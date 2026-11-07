# vivat

Infrastructure for Sct. Omega Broderskab: what runs on the k3s cluster, deployed by
[Argo CD](https://argo-cd.readthedocs.io) from git. A change to the cluster is a pull request
to this repo, and Argo CD applies what `main` says.

```
clusters/
├── prod/       # The production cluster: root app, cluster resources and the services
└── local/      # A k3d cluster on a workstation, for trying changes first
platform/       # Operators the services depend on: cert-manager, CloudNativePG, Phase
cluster/        # Cluster-wide resources such as the Let's Encrypt issuer
```

## How it fits together

Each cluster has a root application that syncs its directory in `clusters/`, itself included.
That directory lists one Argo CD `Application` per service. The service's own manifests stay
in the service's repo, and the `Application` here names the release of them that runs: it pins
the repo to the release tag and sets the images to the same tag.

| Service | Repo | Manifests | Release named in |
|---|---|---|---|
| omegaweb | [omegaweb-next](https://github.com/vevcom/omegaweb-next) | `deploy/overlays/prod` | `clusters/prod/omegaweb.yaml` |

Applications sync in waves: the operators and cluster resources in wave -1, the services in
wave 0. Within a service, its own manifests decide the order, for example the database before
the migrations before the app.

## Releasing a service

Publishing a GitHub release of omegaweb-next builds its images and opens a pull request here
that sets the new tag in `clusters/prod/omegaweb.yaml`. Merging that pull request is the deploy,
and a rollback is a revert of it. Nothing merged to main of a service repo reaches prod until a
release names it, so a change to a service's manifests is tried on `clusters/local` first.

## Adding a service

1. Give the service a `deploy/` directory in its repo with an overlay per cluster, as
   omegaweb-next has. It runs in its own namespace.
2. Add an `Application` for it to `clusters/prod/` and `clusters/local/`, and list it in their
   `kustomization.yaml`. In prod, pin `targetRevision` to a release tag and set the images under
   `kustomize.images`, as `omegaweb.yaml` does.
3. Secrets come from the service's own Phase app through a `PhaseSecret` in its manifests.
4. A private repo needs credentials Argo CD can read it with. A GitHub App on the vevcom org,
   stored as a `repo-creds` secret for `https://github.com/vevcom`, covers every repo at once.

Third-party software, such as Grafana, is installed from its upstream Helm chart like the
operators in `platform/`, with values kept in this repo.

## Setting up a cluster

Install k3s, then Argo CD, then apply the cluster's directory once:

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm install argocd argo/argo-cd --version 10.10.5 --namespace argocd --create-namespace --wait
kubectl apply -k clusters/prod
```

Argo CD then installs the operators and every service. Secrets that cannot come from git are
made by hand once per cluster, as each service's README describes. For omegaweb those are the
Phase service token and the pull secret for its images.

## Trying it locally

omegaweb-next's `deploy/k3d/up.sh --argocd` sets up a k3d cluster that follows
`clusters/local`. `clusters/local/omegaweb.yaml` names the branch of omegaweb-next it deploys.
