# Getting started

This guide covers two paths:

- **A. Local lab.** Minikube or Vagrant on your workstation, for trying changes.
- **B. Bare metal.** The real k3s cluster on Raspberry Pis, managed through CI.

Both paths need the external accounts listed in [0. Before you start](#0-before-you-start).

---

## 0. Before you start

### Tools

```bash
# macOS (Homebrew); use the equivalent packages on Linux
brew install go-task kubectl helm helmfile ansible openssl
helm plugin install https://github.com/databus23/helm-diff   # required by `helmfile apply`
# Local lab only: minikube (+ Docker) or vagrant (+ VirtualBox)
```

### External accounts and secrets

| What | Why | Where it goes |
| --- | --- | --- |
| **Bitwarden Secrets Manager** organization + project + *machine account* | Source of every in-cluster secret | Token → `BITWARDEN_MA_ACCESS_TOKEN`. Org/project IDs → `core/external-secrets/secret-store/bitwarden/secret-store.yaml` |
| **Cloudflare** domain + Zero Trust tunnel named `fabricator` | External access | Tunnel credentials stored in Bitwarden (see [core-services.md](core-services.md#cloudflared)) |
| **GitHub** PAT with access to this repository | Argo CD repo access and ARC runner registration | Argo CD: Bitwarden. ARC: `GITHUB_ARC_ACCESS_TOKEN` |
| **Private CA** key pair | cert-manager `ca-issuer` | `CERT_MANAGER_CA_TLS_CRT` / `CERT_MANAGER_CA_TLS_KEY` |

### Generate the CA

```bash
mkdir -p core/cert-manager/ca
openssl genrsa -out core/cert-manager/ca/tls.key 2048
openssl req -x509 -nodes -new -sha256 -days 365 \
  -key core/cert-manager/ca/tls.key \
  -extensions v3_ca \
  -out core/cert-manager/ca/tls.crt \
  -config core/cert-manager/openssl.cnf
```

`tls.crt` and `tls.key` are listed in `.gitignore` and must never be committed.

---

## A. Local lab

### A.1 Configure

```bash
cp .env.default .env
```

| Variable | Default | Meaning |
| --- | --- | --- |
| `K8S_ENGINE` | `minikube` | `minikube` or `k3s` (Vagrant) |
| `MINIKUBE_CLUSTER_MEMORY` / `_CPUS` | `10240` / `4` | Resources for the Minikube node |
| `MINIKUBE_MACHINE_MOUNTPATH` → `MINIKUBE_CLUSTER_MOUNTPATH` | `/tmp/minikube` → `/data` | Host directory mounted into the node |
| `K3S_SERVER_MEMORY` / `_CPUS` | `2048` / `1` | Vagrant server VM (`homelab-1`, 192.168.56.10) |
| `K3S_AGENT_MEMORY` / `_CPUS` | `4096` / `1` | Vagrant agent VMs (`homelab-2/3`, .11/.12) |
| `LAB_DEPLOY_CORE_SERVICES_ON_STARTUP` | `false` | Reserved; not read by any task yet |

`.env` takes precedence over `.env.default`, and it is git-ignored.

### A.2 Lifecycle

```bash
task lab-on       # start the cluster
task lab-status   # show its status
task lab-off      # stop it (state is kept)
task lab-delete   # destroy it
```

- **Minikube** starts a single Docker-driver node and then runs `kubectl ctx minikube`. That
  command needs the [kubectx](https://github.com/ahmetb/kubectx) plugin; if you don't have it, run
  `kubectl config use-context minikube` instead.
- **k3s (Vagrant)** only creates three Ubuntu 22.04 VMs on a private network. It does **not**
  install k3s. To install it, point an Ansible inventory at `192.168.56.10-12` and follow
  [B.2](#b2-install-k3s), or install k3s on the VMs by hand.

### A.3 Install the core services

```bash
export CERT_MANAGER_CA_TLS_CRT="$(cat core/cert-manager/ca/tls.crt)"
export CERT_MANAGER_CA_TLS_KEY="$(cat core/cert-manager/ca/tls.key)"
export BITWARDEN_MA_ACCESS_TOKEN="<token>"

cd core && helmfile repos && cd ..
task core
```

Helmfile installs the releases in this order, waiting up to 600 s for each:
**cert-manager → external-secrets → argo-cd → cloudflared**. Each release's `postsync` hook then
applies its kustomization: the CA issuer, the secret store, the repo secret and app-of-apps, and the
tunnel secret.

> On a local lab you may not want the Cloudflare tunnel. Comment out
> `cloudflared/helmfile.yaml` in `core/helmfile.yaml` (and leave the change uncommitted), or
> run `helmfile -l name!=cloudflared apply`.

---

## B. Bare-metal cluster

### B.1 Prepare the nodes

1. Flash Raspberry Pi OS Lite (64-bit) or Ubuntu Server arm64 onto each node.
2. Give each node a static IP or a DHCP reservation.
3. Set up SSH key access for the Ansible user, with passwordless `sudo`.
4. Edit `provisioning/inventories/bare-metal.yml`:
   - Hosts go under `server` (one node) and `agent` (the rest).
   - Set `ansible_user`, `k3s_version` (currently `v1.31.12+k3s1`) and `cluster_context`.
   - `token` is the k3s join token. Use a fresh random value and keep it out of Git if you can,
     for example with `ansible-vault` or `--extra-vars`. See
     [troubleshooting.md](troubleshooting.md#known-issues).
5. Update `remote_user` in `provisioning/playbook.yml` if your user is different.

### B.2 Install k3s

```bash
cd provisioning
ansible-galaxy collection install -r collections/requirements.yml
task cluster          # playbook.yml --tags prepare_cluster
```

This runs the `prereq`, `airgap` and `raspberrypi` roles on every host, then `k3s_server`
and `k3s_agent`. The server role merges the kubeconfig into `~/.kube/config` on your machine
under the context `k3s-ansible`.

```bash
kubectl --context k3s-ansible get nodes -o wide
```

Optional: set aside a node for local LLMs.

```bash
kubectl label node <node> dedicated=ai_agent
kubectl taint node <node> dedicated=ai_agent:NoSchedule
```

### B.3 Install the CI runners (ARC)

```bash
export GITHUB_ARC_ACCESS_TOKEN=<github PAT that can register runners on the repo>
task core-services    # playbook.yml --tags prepare_core_service
```

This installs `core/arc` into the `arc` namespace. It contains the scale-set controller, the
`arc-fabricator` runner scale set (0–3 runners), and the `arc-runner` ServiceAccount bound to
`cluster-admin`.

`task provision` runs both B.2 and B.3.

### B.4 Configure GitHub

1. **Secrets** (*Settings → Secrets and variables → Actions*):
   `CERT_MANAGER_CA_TLS_CRT`, `CERT_MANAGER_CA_TLS_KEY`, `BITWARDEN_MA_ACCESS_TOKEN`.
   Paste the raw PEM or token; the templates do the base64 encoding.
2. **Environment** (*Settings → Environments*): create `production` with **required reviewers**.

### B.5 Deploy the core services

Open a PR to `master`. The `validate` job renders the core manifests. When the PR is merged, the
`deploy` job waits for approval and then runs `helmfile apply` from inside the cluster.

You can also run `task core` from your machine with the three variables exported, as in A.3.

### B.6 Let Argo CD take over

Argo CD's `postsync` hook creates the `bootstrap` Application. It points at
`bootstrap/chart` on branch **`lab`**, with values from `bootstrap/bootstraps.yaml`.

```bash
# Initial admin password
kubectl -n argo get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d

# UI (no ingress is defined; port-forward or add a Cloudflare route)
kubectl -n argo port-forward svc/argo-cd-server 8080:80
```

Auto-sync is **off** by default (`syncConfig.autoSync.enabled: false`). Sync `bootstrap` first,
then sync each child application from the UI or with `argocd app sync <name>`.

---

## Verify

```bash
kubectl get pods -A
kubectl get clustersecretstore bitwarden-secretsmanager      # STATUS should be Valid
kubectl get externalsecrets -A                               # every row should be SecretSynced
kubectl get clusterissuer ca-issuer                          # READY=True
kubectl -n argo get applications
kubectl -n cloudflare logs deploy/cloudflared | grep -i registered
```

## Next steps

- Turn applications on or off: [gitops.md](gitops.md)
- See which applications are available: [platforms.md](platforms.md)
