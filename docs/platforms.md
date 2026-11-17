# Platform catalog

This page lists every wrapper chart under `platforms/`. **Enabled** reflects
[`bootstrap/bootstraps.yaml`](../bootstrap/bootstraps.yaml) at the time of writing. Unless noted
otherwise, each app is deployed to a namespace with the same name as the app.

Legend: ✅ enabled · ⏸ defined but disabled · 🚫 commented out of the catalog

## AI: `platforms/ai/`

Details are in [ai-stack.md](ai-stack.md).

| App | Enabled | Upstream chart | Version | Notes |
| --- | --- | --- | --- | --- |
| `litellm` | ✅ | `litellm-helm` (oci://ghcr.io/berriai) | 1.108.0 | OpenAI-compatible LLM gateway (image v1.85.1). Its DB is in CNPG |
| `ollama` | ✅ | `unichart` | 1.0.1 | Local models (ollama 0.23.2). Pinned to the `dedicated=ai_agent` node |
| `open-webui` | ✅ | `open-webui` (helm.openwebui.com) | 14.5.0 | Chat UI (0.9.5) at `chat.thefabricator.work` |
| `jarvis` | ✅ | `unichart` | 1.0.1 | Custom agent service `sondn1/jarvis:v1.0.1` |
| `mcp` | ✅ | `kubernetes-mcp-server` (oci://ghcr.io/containers/charts) | 0.1.0 | Kubernetes MCP server with read-only `apps/*` access |
| `openclaw` | ⏸ | `unichart` | 1.0.1 | OpenClaw agent gateway at `agent.thefabricator.work` |

## Computation: `platforms/computation/`

| App | Enabled | Upstream chart | Version | Notes |
| --- | --- | --- | --- | --- |
| `airflow` | ⏸ | Bitnami `airflow` | 22.7.2 | Includes a StatsD exporter mapping for Prometheus |
| `jupyterhub` | ⏸ | `jupyterhub` + `enterprise-gateway` (vendored .tgz) | 4.3.2 / 3.2.3 | Gateway enabled with `enterprise-gateway.enabled` |
| `nifi` | ⏸ | `nifi` (vendored source chart) | 0.1.0 | Ships full NiFi config, a ServiceMonitor and a PrometheusRule |

## Databases: `platforms/databases/`

| App | Enabled | Upstream chart | Version | Notes |
| --- | --- | --- | --- | --- |
| `elasticsearch` | ⏸ | Bitnami `elasticsearch` + `kibana` | 21.3.16 / 11.2.20 | Kibana enabled with `kibana.enabled` |
| `mongodb` | ⏸ | `mongodb-kubernetes` operator | 1.8.1 | |
| `mysql` | ⏸ | Bitnami `mysql` | 11.1.16 | |
| `opensearch` | ⏸ | `opensearch` + `opensearch-dashboards` | 3.6.0 | See [known issues](troubleshooting.md#known-issues) |
| `postgresql` | 🚫 | Bitnami `postgresql` + vendored `pgbouncer` | 15.5.24 | Replaced by **CNPG** (Bitnami charts are deprecated) |
| `qdrant` | ⏸ | `qdrant` | 1.17.0 | Vector database |
| `redis` | ⏸ | Bitnami `redis` | 20.1.3 | |

## Monitoring: `platforms/monitoring/`

Details are in [observability.md](observability.md).

| App | Enabled | Upstream chart | Version | Notes |
| --- | --- | --- | --- | --- |
| `observability` | ✅ | `kube-prometheus-stack` | 87.2.0 | Prometheus, Alertmanager, Grafana, node-exporter, kube-state-metrics |
| `loki` | ⏸ | Bitnami `grafana-loki` | 4.6.15 | Log store that Fluentd writes to |

## Operators: `platforms/operators/`

| App | Enabled | Upstream chart | Version | Notes |
| --- | --- | --- | --- | --- |
| `cnpg` | ✅ | `cloudnative-pg` | 0.29.1 (operator 1.30.1) | Also creates the shared **`Cluster/cnpg`** (PostgreSQL 18.6, 3 Gi) and its `Database` CRs |
| `fluent` | ⏸ | `fluent-operator` (vendored .tgz) | 3.1.0 | `ClusterOutput` to Loki |
| `spark` | ⏸ | `spark-operator` (kubeflow) | 2.0.0-rc.0 | Includes an example `SparkApplication` (spark-pi) |
| `strimzi` | ⏸ | `strimzi-kafka-operator` + `schema-registry` + `akhq` | 0.43.0 | KRaft `Kafka` with a node pool of 3 |

### The shared PostgreSQL cluster (CNPG)

`platforms/operators/cnpg/templates/` creates one `Cluster` named `cnpg` in the `cnpg` namespace.
Platform apps use it instead of running their own databases.

- Image `ghcr.io/cloudnative-pg/postgresql:18.6-system-trixie`, 1 instance, 3 Gi storage, with
  `pg_stat_statements` and `auto_explain` set up.
- The superuser credentials (`cnpg-superuser-secret`) and an init SQL script (`cnpg-initdb-secret`,
  which creates the roles) come from Bitwarden.
- `Database` CRs: `airflow`, `jupyterhub`, `keycloak`, `argo_workflows`, `ucdb`, `litellm`. Each
  has `databaseReclaimPolicy: retain`.
- The operator only watches its own namespace (`WATCH_NAMESPACE: cnpg`).

Consumers connect to `cnpg-rw.cnpg.svc:5432`. LiteLLM gets its endpoint and credentials from
Bitwarden.

To add a database, add a `Database` CR to `databases.yaml`, and make sure the owner role exists by
adding it to the init SQL in Bitwarden or creating it by hand.

## Security: `platforms/security/`

| App | Enabled | Upstream chart | Version | Notes |
| --- | --- | --- | --- | --- |
| `keycloak` | ⏸ | Bitnami `keycloak` | 22.2.4 | Intended to use the CNPG `keycloak` database |
| `vault` | ⏸ | Bitnami `vault` | 1.4.22 | The unseal procedure is still TODO |

## Storage: `platforms/storage/`

| App | Enabled | Upstream chart | Version | Notes |
| --- | --- | --- | --- | --- |
| `minio` | ⏸ | Bitnami `minio` | 14.7.7 | S3-compatible object store |
| `seaweedfs` | ⏸ | `seaweedfs` | 4.24.0 | Distributed object and file store |

## Streaming: `platforms/streaming/`

| App | Enabled | Upstream chart | Version | Notes |
| --- | --- | --- | --- | --- |
| `kafka` | ⏸ | Bitnami `kafka` + `schema-registry` + `akhq` | 30.1.1 | Use either this or Strimzi, not both |
| `debezium` | ⏸ | *raw manifests* | — | Strimzi `KafkaConnect` (`sondn1/debezium:2.1.3`) plus a sample Postgres connector. Not a Helm chart |

## Warehouse: `platforms/warehouse/`

| App | Enabled | Upstream chart | Version | Notes |
| --- | --- | --- | --- | --- |
| `clickhouse` | ⏸ | Bitnami `clickhouse` | 6.2.23 | |

## Utilities: `platforms/ultilities/`

| App | Enabled | Upstream chart | Version | Notes |
| --- | --- | --- | --- | --- |
| `headlamp` | ✅ | `headlamp` (kubernetes-sigs) | 0.46.0 | Kubernetes UI with `cluster-admin`, authenticated by Cloudflare Access |
| `dbgate` | ✅ | `unichart` | 1.0.1 | Web database client (7.3.4) on :3000 |
| `toolbox` | ✅ | `unichart` | 1.0.1 | `sondn1/toolbox:0.0.1` debug pod with a ServiceAccount |
| `code-server` | ⏸ | vendored `code-server` | 3.25.3 | VS Code in the browser |
| `localstack` | ⏸ | `localstack` | 0.6.17 | Local AWS API emulator |
| `metrics-server` | ⏸ | Bitnami `metrics-server` | 7.2.14 | k3s already ships metrics-server |
| `zookeeper` | ⏸ | Bitnami `zookeeper` | 13.4.13 | Only needed by components that aren't on KRaft |

> **About Bitnami charts.** Many disabled apps still use `oci://registry-1.docker.io/bitnamicharts`.
> Bitnami has moved most free images and charts to a legacy, unmaintained repository. Before you
> re-enable one of these apps, check that the image still exists for `arm64`, or move it to the
> project's official chart, as was done for PostgreSQL → CNPG.
