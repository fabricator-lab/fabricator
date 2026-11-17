# Troubleshooting & known issues

## Troubleshooting

### `required env var 'X' is not set`

Helmfile's `requiredEnv` stops templating when a bootstrap secret is missing. Export all three
variables before running `task core`, and quote them because the PEM spans several lines:

```bash
export CERT_MANAGER_CA_TLS_CRT="$(cat core/cert-manager/ca/tls.crt)"
export CERT_MANAGER_CA_TLS_KEY="$(cat core/cert-manager/ca/tls.key)"
export BITWARDEN_MA_ACCESS_TOKEN="..."
```

In CI, check that the repository secrets exist and are spelled exactly as above.

### Chart signature verification fails

`core/helmfile.yaml` sets `helmDefaults.verify: true`. If an upstream chart is published without a
`.prov` provenance file, both `helmfile template` and `helmfile apply` fail. You can obtain
provenance for that chart, or set `verify: false` on that release only. Prefer the per-release
override over turning verification off for every release.

### `ClusterSecretStore` is not `Valid` / `ExternalSecret` shows `SecretSyncedError`

```bash
kubectl get clustersecretstore bitwarden-secretsmanager -o yaml | yq .status
kubectl -n external-secrets logs deploy/external-secrets
kubectl -n external-secrets logs deploy/bitwarden-sdk-server
kubectl -n external-secrets get certificate bitwarden-sdk-server
```

Common causes:

- The machine-account token has expired, or the account can't access the project.
- The `organizationID` or `projectID` is wrong after a fork.
- The SDK server certificate isn't `Ready` yet because cert-manager or `ca-issuer` isn't ready.
- A `remoteRef.key` refers to a secret ID that doesn't exist in the project.

### cloudflared pods crash right after the first install

The tunnel secret is applied in a postsync hook, and ESO needs a moment to fill it. Check that
`kubectl -n cloudflare get externalsecret cloudflare-secret` shows `SecretSynced`, then run
`kubectl -n cloudflare rollout restart deploy/cloudflared`.

### Argo CD can't reach the repository

Check that `argocd-github-fabricator-secret` exists in `argo` and has the label
`argocd.argoproj.io/secret-type=repository`. The GitHub token must be able to read the repository.

### An app doesn't show up in Argo CD

- Check that `enabled: true` is set in `bootstrap/bootstraps.yaml`, **on the `lab` branch**.
- Sync the `bootstrap` Application. Auto-sync is off by default.
- Render the chart locally to catch template errors:

  ```bash
  helm template bootstrap bootstrap/chart -f bootstrap/bootstraps.yaml | less
  ```

### CI jobs stay queued

- Check that the runners exist: `kubectl -n arc get autoscalingrunnerset,ephemeralrunner`.
- Check the controller logs: `kubectl -n arc logs deploy -l app.kubernetes.io/part-of=gha-rs-controller`.
- The workflow uses `runs-on: arc-fabricator`, which must match `runnerScaleSetName`.
- The `deploy` job waits for approval on the `production` environment.

### Ollama is `Pending`

No node has both the `dedicated=ai_agent` label and the matching taint, or the node doesn't have
enough free memory for the 2 Gi request. See
[getting-started.md](getting-started.md#b2-install-k3s).

### General

```bash
kubectl get pods -A | grep -vE 'Running|Completed'
kubectl -n <ns> describe pod <pod>
kubectl -n <ns> logs <pod> --previous
kubectl get events -A --sort-by=.lastTimestamp | tail -30
```

---

## Known issues

These were found while reviewing the repository. They are recorded here so they can be fixed
deliberately.

### Security

| Issue | Where | Suggestion |
| --- | --- | --- |
| The k3s join `token` is committed in plaintext | `provisioning/inventories/bare-metal.yml` | Rotate it, then move it to `ansible-vault`, an environment lookup, or `--extra-vars` |
| The ARC runner ServiceAccount is `cluster-admin` | `core/arc/templates/runner-rbac.yaml` | Accepted trade-off. A narrower ClusterRole is possible if the core CRDs are installed separately |
| Headlamp uses a `cluster-admin` ServiceAccount token for every user | `platforms/ultilities/headlamp/values.yaml` | Safe only behind Cloudflare Access. Consider turning on the OIDC config that `oidc-secret` already prepares |
| The MCP server image uses the `latest` tag | `platforms/ai/mcp/kubernetes-mcp-server.yaml` | Pin a version |

### Correctness

| Issue | Where | Effect |
| --- | --- | --- |
| The `jarvis` chart contains `templates/configmap.yaml`, which renders `ConfigMap litellm-config` **in namespace `litellm`** from `files/config.yaml`, but the jarvis chart has no `files/` directory | `platforms/ai/jarvis/templates/configmap.yaml` | Two Argo apps own the same ConfigMap, and Jarvis can overwrite LiteLLM's model config with an empty one. It looks copied from the litellm chart, so it can probably be deleted |
| `opensearch-dashboards` uses `condition: pgbouncer.enabled`, but the catalog sets `opensearchDashboards.enabled` | `platforms/databases/opensearch/Chart.yaml` | Dashboards are never installed. Change the condition to `opensearch-dashboards.enabled` |
| The `clickhouse` catalog entry sets `schema-registry.enabled` and `akhq.enabled`, which aren't dependencies of that chart | `bootstrap/bootstraps.yaml` | No effect. Looks copied from the kafka entry |
| `debezium` has no `Chart.yaml` or `values.yaml`, but its catalog entry lists `values.yaml` | `platforms/streaming/debezium/` | Enabling it fails. Remove `valueFiles` (Argo CD would treat it as plain manifests) or wrap it in a chart. It also points at `databus-kafka-bootstrap.kafka.svc`, which neither Kafka option creates |
| The bootstrap template renders `info` and `ignoreDifferences` from `$app.valueFiles`, and renders `labels` as a list | `bootstrap/chart/templates/application.yaml` | These fields can't be used as documented. Use `$app.info`, `$syncCfg.ignoreDifferences`, and a map for labels |
| Ollama only mounts the `smollm2:135m` Modelfile | `platforms/ai/ollama/values.yaml` | The other three models are in the ConfigMap but are never created |
| `clusters/Taskfile.yml` uses `var:` instead of `vars:` | `clusters/Taskfile.yml` | The `minikube` default is ignored. It works today only because `.env.default` sets `K8S_ENGINE` |
| The `core.yml` comments refer to `docker/arc-runner/Dockerfile`, `core/cert-manager/secrets.yaml` and `core/external-secrets/bitwarden/...` | `.github/workflows/core.yml`, `core/arc/values.yaml` | These paths don't exist. The real files are `core/cert-manager/issuer-secret.yaml` and `core/external-secrets/ma-access-token.yaml`, and the runner Dockerfile isn't in the repository |
| The Vagrant k3s lab only creates VMs | `clusters/k3s/Vagrantfile` | `task lab-on` with `K8S_ENGINE=k3s` doesn't install k3s. There is no provisioner or inventory for 192.168.56.x |
| `LAB_DEPLOY_CORE_SERVICES_ON_STARTUP` isn't used by any task | `.env.default` | Either wire it into `lab-on` or remove it |

### Housekeeping

- Several `Chart.yaml` names don't match their directory. `ai/ollama` is named `toolbox`,
  `ultilities/toolbox` is named `ollama`, `ai/openclaw` is named `jarvis`, and
  `ultilities/zookeeper` is named `strimzi-operator`. This only affects how charts are labelled.
- The directory is spelled `ultilities/` instead of `utilities/`. Renaming it means updating every
  `path:` in `bootstraps.yaml`.
- Many disabled apps depend on Bitnami charts and images that are no longer maintained. See
  [platforms.md](platforms.md).
- `.vscode/settings.json` contains an absolute path from one developer's machine.
