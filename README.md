# k3s cluster backed by Flux

A single [k3s](https://k3s.io/) cluster backed by [Flux](https://toolkit.fluxcd.io/) and [SOPS](https://toolkit.fluxcd.io/guides/mozilla-sops/).

Kubernetes cluster using the [GitOps](https://www.weave.works/blog/what-is-gitops-really) tool [Flux](https://toolkit.fluxcd.io/). The Git repository is the driving the state of the Kubernetes cluster. In addition with the help of the [Flux SOPS integration](https://toolkit.fluxcd.io/guides/mozilla-sops/) secrets can be commited to the repo GPG encrypted.

## :wave:&nbsp; Introduction

The following components will are installed in the [k3s](https://k3s.io/) cluster.

- [flux](https://toolkit.fluxcd.io/)
- [metallb](https://metallb.universe.tf/)
- [ingress-nginx](https://kubernetes.github.io/ingress-nginx/)
- [cert-manager](https://cert-manager.io/) with Cloudflare DNS challenge
- [longhorn]()
- [rancher]()
- [homer](https://github.com/bastienwirtz/homer)

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
│   └── system-upgrade
├── base
│   └── flux-system
├── core
│   ├── cert-manager
│   ├── metallb-system
│   ├── namespaces
│   └── system-upgrade
└── crds
    └── cert-manager
```

## :rocket:&nbsp; Lets go!

### :closed_lock_with_key:&nbsp; Setting up GnuPG keys

:round_pushpin: Here we will create a personal and a Flux GPG key. Using SOPS with GnuPG allows us to encrypt and decrypt secrets.

1. Create a Personal GPG Key, password protected, and export the fingerprint. It's **strongly encouraged** to back up this key somewhere safe so you don't lose it.

```sh
export GPG_TTY=$(tty)
export PERSONAL_KEY_NAME="First name Last name (location) <email>"

gpg --batch --full-generate-key <<EOF
Key-Type: 1
Key-Length: 4096
Subkey-Type: 1
Subkey-Length: 4096
Expire-Date: 0
Name-Real: ${PERSONAL_KEY_NAME}
EOF

gpg --list-secret-keys "${PERSONAL_KEY_NAME}"
```

2. Create a Flux GPG Key and export the fingerprint

```sh
export GPG_TTY=$(tty)
export FLUX_KEY_NAME="Cluster name (Flux) <email>"

gpg --batch --full-generate-key <<EOF
%no-protection
Key-Type: 1
Key-Length: 4096
Subkey-Type: 1
Subkey-Length: 4096
Expire-Date: 0
Name-Real: ${FLUX_KEY_NAME}
EOF

gpg --list-secret-keys "${FLUX_KEY_NAME}"
```

### :small_blue_diamond:&nbsp; GitOps with Flux

:round_pushpin: Here we will be installing [flux](https://toolkit.fluxcd.io/) after some quick bootstrap steps.

1. Verify Flux can be installed

```sh
flux check --pre
```

2. Pre-create the `flux-system` namespace

```sh
kubectl create namespace flux-system --dry-run=client -o yaml | kubectl apply -f -
```

3. Add the Flux GPG key in-order for Flux to decrypt SOPS secrets

```sh
gpg --export-secret-keys --armor "${FLUX_KEY_FP}" |
kubectl create secret generic sops-gpg \
    --namespace=flux-system \
    --from-file=sops.asc=/dev/stdin
```

4. Encrypt `cluster/cluster-secrets.yaml`, `cert-manager/secret.enc.yaml` and `flux-system/flux-secret.yaml` with SOPS

```sh
export GPG_TTY=$(tty)
sops --encrypt --in-place ./cluster/base/cluster-secrets.yaml
sops --encrypt --in-place ./cluster/core/cert-manager/secret.enc.yaml
sops --encrypt --in-place ./cluster/base/flux-system/flux-secret.yaml
```

5. Add the Flux SSH key in-order for Flux to pull private git repositories

```sh
sops -d ./cluster/base/flux-system/flux-secret.yaml | kubectl apply -f -
```

6. **Verify** all the above files are **encrypted** with SOPS

7. Push you changes to git

```sh
git add -A
git commit -m "initial commit"
git push
```

8. Install Flux

:round_pushpin: Due to race conditions with the Flux CRDs you will have to run the below command twice. There should be no errors on this second run.

```sh
kubectl apply --kustomize=./cluster/base/flux-system
```

## :mega:&nbsp; Post installation

### :point_right:&nbsp; Debugging

Manually sync Flux with your Git repository

```sh
flux reconcile source git flux-system
```

Show the health of you kustomizations

```sh
kubectl get kustomization -A
```

Show the health of your main Flux `GitRepository`

```sh
flux get sources git
```

Show the health of your `HelmRelease`s

```sh
flux get helmrelease -A
```

Show the health of your `HelmRepository`s

```sh
flux get sources helm -A
```

### :robot:&nbsp; Automation

- [Renovate](https://www.whitesourcesoftware.com/free-developer-tools/renovate) is a very useful tool that when configured will start to create PRs in your Github repository when Docker images, Helm charts or anything else that can be tracked has a newer version. The configuration for renovate is located [here](./.github/renovate.json5).
- [Flux upgrade schedule](./.github/workflows/flux-schedule.yaml) - workflow to upgrade Flux.
- [Renovate schedule](./.github/workflows/renovate-schedule.yaml) - workflow to annotate `HelmRelease`'s which allows [Renovate](https://www.whitesourcesoftware.com/free-developer-tools/renovate) to track Helm chart versions.

## :handshake:&nbsp; Thanks

Big shout out to [k8s@home](https://github.com/k8s-at-home) for their [k3s-cluster-template](https://github.com/k8s-at-home/template-cluster-k3s) :heart:
