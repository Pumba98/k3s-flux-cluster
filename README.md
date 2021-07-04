# k3s cluster backed by Flux

A single [k3s](https://k3s.io/) cluster backed by [Flux](https://toolkit.fluxcd.io/) and [SOPS](https://toolkit.fluxcd.io/guides/mozilla-sops/).

Kubernetes cluster using the [GitOps](https://www.weave.works/blog/what-is-gitops-really) tool [Flux](https://toolkit.fluxcd.io/). The Git repository is the driving the state of the Kubernetes cluster. In addition with the help of the [Flux SOPS integration](https://toolkit.fluxcd.io/guides/mozilla-sops/) secrets can be commited to the repo GPG encrypted.

## :wave:&nbsp; Software

The following components will are installed in the [k3s](https://k3s.io/) cluster.

| Software                                                     | Purpose                                                |
| ------------------------------------------------------------ | ------------------------------------------------------ |
| [flux](https://toolkit.fluxcd.io/)                           | GitOps Tool managing the cluster                       |
| [metallb](https://metallb.universe.tf/)                      | Bare metal load-balancer                               |
| [ingress-nginx](https://kubernetes.github.io/ingress-nginx/) | Cluster ingress controller                             |
| [cert-manager](https://cert-manager.io/)                     | letsencrypt certificates with Cloudflare DNS challenge |
| [longhorn](https://longhorn.io/)                             | Persistent Storage                                     |
| [rancher](https://rancher.com/products/rancher/)             | Kubernetes Management dashboard                        |
| [homer](https://github.com/bastienwirtz/homer)               | Static dashboard for the cluster                       |

## :wrench:&nbsp; Tools

:round_pushpin: You need to install the required CLI tools listed below on your workstation.

| Tool                                                                                               | Purpose                                                             | Minimum version |
| -------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------- | :-------------: |
| [kubectl](https://kubernetes.io/docs/tasks/tools/)                                                 | Allows you to run commands against Kubernetes clusters              |    `1.21.0`     |
| [flux](https://toolkit.fluxcd.io/)                                                                 | Operator that manages your k8s cluster based on your Git repository |    `0.12.3`     |
| [SOPS](https://github.com/mozilla/sops)                                                            | Encrypts k8s secrets with GnuPG                                     |     `3.7.1`     |
| [GnuPG](https://gnupg.org/)                                                                        | Encrypts and signs your data                                        |    `2.2.27`     |
| [pre-commit](https://github.com/pre-commit/pre-commit)                                             | Runs checks during `git commit`                                     |    `2.12.0`     |
| [VSCode SOPS](https://marketplace.visualstudio.com/items?itemName=signageos.signageos-vscode-sops) | automatically decrypt/encrypt your SOPS secrets in VSCode           |     `0.4.0`     |

### :warning:&nbsp; pre-commit

Install [pre-commit](https://pre-commit.com/) and the pre-commit hooks that come with this repository.
[sops-pre-commit](https://github.com/k8s-at-home/sops-pre-commit) will check to make sure you are not by accident commiting your secrets un-encrypted.

After pre-commit is installed on your machine run:

```sh
pre-commit install-hooks
```

## :open_file_folder:&nbsp; Repository structure

The Git repository contains the following directories under `cluster` and are ordered below by how Flux will apply them.

- **base** directory is the entrypoint to Flux
- **crds** directory contains custom resource definitions (CRDs) that need to exist globally in your cluster before anything else exists
- **core** directory (depends on **crds**) are important infrastructure applications (grouped by namespace) that should never be pruned by Flux
- **apps** directory (depends on **core**) is where your common applications (grouped by namespace) could be placed, Flux will prune resources here if they are not tracked by Git anymore

```
cluster
├── apps
│   ├── default
│   ├── networking
│   └ ...
├── base
│   └── flux-system
├── core
│   ├── cert-manager
│   ├── metallb-system
│   ├── namespaces
│   └ ...
└── crds
    ├── cert-manager
    └ ...
```

## :rocket:&nbsp; Lets go!


:round_pushpin: Some notes how to install [flux](https://toolkit.fluxcd.io/) into the cluster.

1. Encrypt secrets with SOPS

```sh
export GPG_TTY=$(tty)
sops --encrypt --in-place ./cluster/base/cluster-secrets.yaml
```

2. Pre-create the `flux-system` namespace

```sh
kubectl create namespace flux-system --dry-run=client -o yaml | kubectl apply -f -
```

3. Add the Flux GPG key in-order for Flux to decrypt SOPS secrets

```sh
gpg --export-secret-keys --armor "F218E6030557131BFE2B62AF9EB6C6C54B075E78" |
kubectl create secret generic sops-gpg \
    --namespace=flux-system \
    --from-file=sops.asc=/dev/stdin
```

4. Add the Flux SSH key in-order for Flux to pull private git repositories

```sh
sops -d ./cluster/base/flux-system/flux-secret.yaml | kubectl apply -f -
```

5. Push everything (**Verify** all secrets are **encrypted** with SOPS!)

6. Install Flux

```sh
kubectl apply --kustomize=./cluster/base/flux-system
```

:round_pushpin: Due to race conditions with the Flux CRDs you will have to run the below command twice. There should be no errors on this second run.


## :robot:&nbsp; Automation

- [Renovate](https://www.whitesourcesoftware.com/free-developer-tools/renovate) is a very useful tool that when configured will start to create PRs in your Github repository when Docker images, Helm charts or anything else that can be tracked has a newer version. The configuration for renovate is located [here](./.github/renovate.json5).
- [Flux upgrade schedule](./.github/workflows/flux-schedule.yaml) - workflow to upgrade Flux.
- [Renovate schedule](./.github/workflows/renovate-schedule.yaml) - workflow to annotate `HelmRelease`'s which allows [Renovate](https://www.whitesourcesoftware.com/free-developer-tools/renovate) to track Helm chart versions.

## :handshake:&nbsp; Thanks

Big shout out to [k8s@home](https://github.com/k8s-at-home) for their [k3s-cluster-template](https://github.com/k8s-at-home/template-cluster-k3s) :heart:
