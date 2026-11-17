# Fabricator documentation

Start with the [project README](../README.md) for an overview. These pages go into more detail.

1. **[Architecture](architecture.md)**: how the pieces fit together, the order things are deployed
   in, how traffic and secrets flow, and the hardware.
2. **[Getting started](getting-started.md)**: going from nothing to a working cluster, either locally
   or on bare metal.
3. **[Core services](core-services.md)**: the components installed by Helmfile that must exist
   before GitOps can run.
4. **[GitOps & the app catalog](gitops.md)**: the app-of-apps chart, `bootstraps.yaml`, and how
   to add a new application.
5. **[Platform catalog](platforms.md)**: every chart under `platforms/`, with its upstream chart,
   version, and whether it is enabled.
6. **[AI stack](ai-stack.md)**: the LLM gateway, local models, chat UI, agents and MCP.
7. **[Secret management](secret-management.md)**: Bitwarden Secrets Manager and External Secrets.
8. **[CI/CD](cicd.md)**: the GitHub Actions workflow and the ARC runners in the cluster.
9. **[Observability](observability.md)**: metrics, dashboards, alerts and logs.
10. **[Troubleshooting & known issues](troubleshooting.md)**

## Conventions used in these docs

- Paths are relative to the repository root.
- `task <name>` means a target in a `Taskfile.yml`. Run it from the directory that contains that file.
- Public hostnames are under `thefabricator.work`. Replace them with your own domain if you fork.
