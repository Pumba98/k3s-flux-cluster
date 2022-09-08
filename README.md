[//]: # "renovate: githubReleaseVar repo=k3s-io/k3s"
[![k3s](https://img.shields.io/badge/k8s-v1.24.4+k3s1-orange?style=for-the-badge&logo=kubernetes)](https://k3s.io/)
[![flux](https://img.shields.io/badge/GitOps-Flux-blue?style=for-the-badge&logo=git)](https://fluxcd.io/)
[![renovate](https://img.shields.io/badge/renovate-enabled-brightgreen?style=for-the-badge&logo=renovatebot)](https://github.com/renovatebot/renovate)
<!-- [![pre-commit](https://img.shields.io/badge/pre--commit-enabled-brightgreen?logo=pre-commit&logoColor=white&style=for-the-badge)](https://github.com/pre-commit/pre-commit) -->

# Single k3s cluster backed by Flux v2

Kubernetes cluster using the [GitOps](https://www.weave.works/blog/what-is-gitops-really) tool [Flux](https://fluxcd.io/).  
The Git repository is the driving the state of the Kubernetes cluster.  
The awesome [Flux SOPS integration](https://toolkit.fluxcd.io/guides/mozilla-sops/) is used to encrypt secrets with gpg.

## :computer:&nbsp; Software

The following components are installed on the [k3s](https://k3s.io/) cluster.

| Software                                                                          | Purpose                                                       |
| --------------------------------------------------------------------------------- | ------------------------------------------------------------- |
| [Flux](https://fluxcd.io)                                                         | GitOps Tool managing the cluster                              |
| [Longhorn](https://longhorn.io)                                                   | Persistent Block Storage Provisioner                          |
| [NGINX Ingress Controller](https://kubernetes.github.io/ingress-nginx)            | Cluster Ingress controller                                    |
| [MetalLB](https://metallb.universe.tf)                                            | Bare metal LoadBalancer                                       |
| [Cert-Manager](https://cert-manager.io)                                           | Letsencrypt certificates with Cloudflare DNS                  |
| [ExternalDNS](https://github.com/kubernetes-sigs/external-dns)                    | Configure Cloudflare DNS Servers                              |
| [kube-vip](https://github.com/kube-vip/kube-vip)                                  | Virtual IP Load-Balancer for Control Plane High Availability  |
| [Kube-Prometheus Stack](https://github.com/prometheus-operator/kube-prometheus)   | Prometheus & Exporters to monitor the cluster                 |
| [Grafana](https://grafana.com)                                                    | Monitoring & Logging Dashboard                                |
| [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager)           | Monitoring Alerts                                             |
| [Grafana Loki](https://grafana.com/oss/loki)                                      | Log aggregation system                                        |
| [System Upgrade Controller](https://github.com/rancher/system-upgrade-controller) | Automated k3s upgrades                                        |
| [Descheduler](https://github.com/kubernetes-sigs/descheduler)                     | Evicts pods to optimize scheduling                            |
| [Authelia](https://www.authelia.com)                                              | SSO & 2FA authentication server for Cluster Web Apps          |
| [Nextcloud](https://nextcloud.com)                                                | File share and collaboration platform                         |
| [Vaultwarden](https://github.com/dani-garcia/vaultwarden)                         | Unofficial Bitwarden compatible server written in Rust        |
| [Firefly-iii](https://www.firefly-iii.org/)                                       | Personal finance manager                                      |
| [Paperless-ngx](https://github.com/paperless-ngx/paperless-ngx)                   | Document management system                                    |
| [Mailu](https://mailu.io/)                                                        | Email stack on kubernetes                                     |
| [Rancher](https://rancher.com/products/rancher)                                   | Kubernetes Management Dashboard                               |
| [Homer](https://github.com/bastienwirtz/homer)                                    | Static dashboard for the cluster applications                 |
| [Pod-Gateway](https://github.com/k8s-at-home/pod-gateway)                         | Route mail traffic through an external gateway                |
| [Goldilocks](https://github.com/FairwindsOps/goldilocks)                          | Utility to help identifying good resource requests and limits |


## :robot:&nbsp; Automation

[Renovate](https://www.whitesourcesoftware.com/free-developer-tools/renovate) Bot makes sure the Cluster is never outdated.

It creates PullRequests when Helm charts or Docker images have newer versions available and even keeps Flux and k3s up-to-date.

## :handshake:&nbsp; Thanks

Big shout out to [k8s@home](https://github.com/k8s-at-home) for their [k3s-cluster-template](https://github.com/k8s-at-home/template-cluster-k3s) and everyone from [awesome-home-kubernetes](https://github.com/k8s-at-home/awesome-home-kubernetes) for the inspiration :heart:

## :open_book:&nbsp; Notes

<details>
    <summary>:round_pushpin: Installation Notes</summary>
<br>
1. :warning:&nbsp; Install pre-commit hooks

```sh
pre-commit install-hooks
```

2. Encrypt all secrets with SOPS

```sh
export GPG_TTY=$(tty)
sops --encrypt --in-place ./cluster/base/cluster-secrets.sops.yaml
```

3. Pre-create the `flux-system` namespace

```sh
kubectl create namespace flux-system --dry-run=client -o yaml | kubectl apply -f -
```

4. Add the Flux GPG key in-order for Flux to decrypt SOPS secrets

```sh
sops -d ./flux-sops-gpg-secret.sops.yaml | kubectl apply -f -
```

5. (Optional) Add the Flux SSH key in-order for Flux to pull private git repositories

```sh
sops -d ./flux-secret.sops.yaml | kubectl apply -f -
```

6. Push everything & Install Flux

```sh
kubectl apply --kustomize=./cluster/base/flux-system
```

:round_pushpin: Due to race conditions with the Flux CRDs run the last command twice. There should be no errors on the second run.

</details>
