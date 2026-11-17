# Core services

Core services are defined in `core/` and installed by Helmfile. Argo CD does not manage them.
The parent [`core/helmfile.yaml`](../core/helmfile.yaml) declares the chart repositories and
shared defaults, then runs the child helmfiles **in order**:

```yaml
helmDefaults:
  wait: true          # block until resources are ready
  force: true         # replace resources whose fields can't be updated in place
  verify: true        # check chart provenance signatures
  createNamespace: true
  timeout: 600

helmfiles:
  - cert-manager/helmfile.yaml
  - external-secrets/helmfile.yaml
  - argo-cd/helmfile.yaml
  - cloudflared/helmfile.yaml
```

Every child release follows the same pattern. It installs the upstream chart with
`fullnameOverride`, then a `postsync` hook runs `kubectl apply -k` on a small kustomization that
holds the CRs which need the chart's CRDs to exist first.

| Release | Namespace | Chart | Version | Postsync kustomization |
| --- | --- | --- | --- | --- |
| cert-manager | `cert-manager` | `jetstack/cert-manager` | v1.16.1 | `issuer/`: `ClusterIssuer ca-issuer` |
| external-secrets | `external-secrets` | `external-secrets/external-secrets` | 0.12.1 | `secret-store/`: Bitwarden cert + `ClusterSecretStore` |
| argo-cd | `argo` | `argo-cd/argo-cd` | 9.0.0 | `resources/`: repo `ExternalSecret` + app-of-apps |
| cloudflared | `cloudflare` | `cloudflare/cloudflare-tunnel` | 0.3.2 | `secret/`: tunnel credentials `ExternalSecret` |
| arc *(installed by Ansible)* | `arc` | local `core/arc` | 1.0.0 (ARC 0.13.1) | — |

---

## cert-manager

`core/cert-manager/`

- Controller, webhook, cainjector and startupapicheck all run image **v1.16.1**, with one replica
  each. CRDs are installed and kept on uninstall (`crds.keep: true`). Leader election uses
  `kube-system`.
- **The CA secret is not stored on disk.** `issuer-secret.yaml` is rendered through `extraObjects`
  with `tpl`. It creates the Secret `fabricator.local-ca` from `requiredEnv "CERT_MANAGER_CA_TLS_CRT"`
  and `"CERT_MANAGER_CA_TLS_KEY"`, so templating fails immediately if either variable is missing.
- `issuer/issuer-ca.yaml` creates `ClusterIssuer/ca-issuer`, backed by that secret.
- `openssl.cnf` holds the `v3_ca` extensions used to generate the CA. See
  [getting-started.md](getting-started.md#generate-the-ca).

To issue a certificate from another chart:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
spec:
  issuerRef: { group: cert-manager.io, kind: ClusterIssuer, name: ca-issuer }
  secretName: my-tls
  dnsNames: [my-svc.my-ns.svc.cluster.local]
```

## External Secrets Operator

`core/external-secrets/`

- Image **v0.12.1**. CRDs for `ClusterExternalSecret`, `ClusterSecretStore`, `ClusterGenerator`
  and `PushSecret` are enabled.
- The **Bitwarden SDK server** sidecar chart is enabled (`bitwarden-sdk-server.enabled: true`).
  ESO calls it over HTTPS on `:9998`.
- `ma-access-token.yaml` is rendered via `extraObjects` and creates `Secret/bitwarden-access-token`
  from `requiredEnv "BITWARDEN_MA_ACCESS_TOKEN"`.
- `secret-store/bitwarden/certificate.yaml` issues the SDK server certificate from `ca-issuer`.
  It is valid for 90 days and renewed 15 days before expiry. This is why cert-manager must be
  installed first.
- `secret-store/bitwarden/secret-store.yaml` creates `ClusterSecretStore/bitwarden-secretsmanager`.
  It points at bitwarden.com, a fixed organization and project, and trusts `ca.crt` from the SDK
  server certificate.

How it is used is described in [secret-management.md](secret-management.md).

## Argo CD

`core/argo-cd/`

- Chart 9.0.0 in namespace **`argo`**, with `fullnameOverride: argo-cd`. Each component (server,
  repo-server, application-controller, redis, dex) runs one replica.
- `config.yaml` is read into `configs.cm` and sets:
  - `admin.enabled: true`, `exec.enabled: false`, `statusbadge.enabled: false`
  - `timeout.reconciliation: 180s`
  - The upstream recommended `ignoreResourceUpdates` and `resource.exclusions` (Endpoints, Leases,
    CertificateRequests, Cilium and Kyverno reports, and similar). This reduces controller churn.
- `resources/repositories/github-fabricator.yaml` is an `ExternalSecret` that produces a Secret
  labelled `argocd.argoproj.io/secret-type: repository`. It holds the GitHub username and token for
  `https://github.com/sondn98/fabricator.git`.
- `resources/app-of-apps.yaml` is the root `Application/bootstrap`. It renders `bootstrap/chart` at
  `targetRevision: lab` with `../bootstraps.yaml`. See [gitops.md](gitops.md).

## cloudflared

`core/cloudflared/`

- Runs the `cloudflare/cloudflared:2024.8.3` image with 2 replicas, connected to the tunnel named
  **`fabricator`**.
- The `ExternalSecret` `cloudflare-secret` builds `credentials.json` from three Bitwarden secrets:
  `AccountTag`, `TunnelID` and `TunnelSecret`.
- Hostname routes are set up in the Cloudflare Zero Trust dashboard, for example
  `chat.thefabricator.work → http://open-webui.open-webui.svc:80`.

> Because the secret is applied in a **postsync** hook, the first install can show cloudflared
> pods failing until the secret exists. They recover by themselves once ESO syncs it.

## Actions Runner Controller (ARC)

`core/arc/`

Ansible installs this chart (see [getting-started.md](getting-started.md#b3-install-the-ci-runners-arc)),
because the runners it creates are what later run `helmfile apply` for everything else.

- Dependencies: `gha-runner-scale-set-controller` and `gha-runner-scale-set` (alias
  **`fabricator`**), both version 0.13.1, from `oci://ghcr.io/actions/actions-runner-controller-charts`.
- The scale set is `arc-fabricator` (used as `runs-on: arc-fabricator`), with 0–3 runners, for
  `https://github.com/sondn98/fabricator`.
- The runner image is `docker.io/sondn1/arc-runner:2.336.1-rc2`. It is a custom image with `helm`,
  `helmfile`, `kubectl` and `helm-diff` preinstalled, so the workflow doesn't install any tools.
- `templates/runner-rbac.yaml` creates `ServiceAccount/arc-runner` and binds it to `cluster-admin`.
  The core releases install CRDs and other cluster-scoped objects, and this is an accepted trade-off
  for a home lab.
- `fabricator.githubConfigSecret.github_token` is injected at install time from
  `GITHUB_ARC_ACCESS_TOKEN`.
- `core/arc/charts/` and `Chart.lock` are git-ignored. Run `helm dependency update` to fetch them.

Details on the workflow are in [cicd.md](cicd.md).
