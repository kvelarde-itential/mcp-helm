# Helm chart for Itential MCP

This repo contains a Helm chart for running the Itential MCP server in Kubernetes. Requires
Helm version `v3.15.0`.

## Itential MCP

The Itential MCP server is a [Model Context Protocol](https://modelcontextprotocol.io/) server
that exposes Itential Platform capabilities as MCP tools. It allows LLM clients such as Claude
to run workflows, manage configurations, query adapters, and monitor platform health by relaying
requests to the Itential Platform REST API.

The server is stateless and is deployed as a Kubernetes Deployment. It does not require any
persistent volumes or external databases. The chart deploys the MCP server container along with
a Kubernetes Service, an optional Ingress, and a Secret for platform credentials.

### Before You Deploy

The following values must be set before installing the chart. The chart will install without
them but the MCP server will be unable to connect to Itential Platform.

| Value | Description |
|:------|:------------|
| `mcp.platform.host` | Hostname or IP address of the Itential Platform instance the MCP server will connect to. |
| `credentials.platformUser` | Username for basic auth against Itential Platform. Set this or the OAuth2 credentials below. |
| `credentials.platformPassword` | Password for basic auth against Itential Platform. |
| `credentials.platformClientId` | OAuth 2.0 client ID. Use instead of basic auth credentials if your platform uses OAuth 2.0. |
| `credentials.platformClientSecret` | OAuth 2.0 client secret. Use instead of basic auth credentials if your platform uses OAuth 2.0. |
| `image.tag` | The MCP server image tag to deploy. Defaults to the chart `appVersion` (`0.12.1`) if not set. |

If you prefer to manage platform credentials outside of the chart, create a Kubernetes Secret
manually and reference it with `credentials.secretName`. When `credentials.secretName` is set,
the chart will not create a Secret and will instead reference the one you provide.

### Usage

Install the MCP server using the defaults in `values.yaml`:

```bash
helm install mcp . -f values.yaml
```

Install with a specific image tag:

```bash
helm install mcp . -f values.yaml --set image.tag=v0.12.1
```

Install with platform credentials passed directly on the command line:

```bash
helm install mcp . -f values.yaml \
  --set mcp.platform.host=iap.example.com \
  --set credentials.platformUser=admin \
  --set credentials.platformPassword=supersecret
```

Install referencing an existing credentials Secret:

```bash
helm install mcp . -f values.yaml \
  --set mcp.platform.host=iap.example.com \
  --set credentials.secretName=my-mcp-credentials
```

### Requirements & Dependencies

The MCP server has no Helm chart dependencies. The only external requirement is a running
Itential Platform instance that the MCP server can reach over the network.

#### Secrets

The chart can manage a credentials Secret for you, or you can bring your own.

##### Chart-managed Secret

When `credentials.secretName` is empty (the default), the chart creates a Secret named
`<release-name>-mcp-credentials` from the `credentials.*` values. Provide either basic auth
credentials or OAuth 2.0 credentials depending on how your Itential Platform is configured.

| Key | Description | Required? |
|:----|:------------|:----------|
| `credentials.platformUser` | Basic auth username for Itential Platform. | One auth method is required |
| `credentials.platformPassword` | Basic auth password for Itential Platform. | One auth method is required |
| `credentials.platformClientId` | OAuth 2.0 client ID for Itential Platform. | One auth method is required |
| `credentials.platformClientSecret` | OAuth 2.0 client secret for Itential Platform. | One auth method is required |

##### Bring Your Own Secret

Set `credentials.secretName` to the name of an existing Kubernetes Secret. The chart will
skip Secret creation and mount the provided secret via `envFrom`. The secret must contain the
relevant `ITENTIAL_MCP_PLATFORM_*` keys for whichever auth method you are using:

| Secret Key | Description |
|:-----------|:------------|
| `ITENTIAL_MCP_PLATFORM_USER` | Basic auth username |
| `ITENTIAL_MCP_PLATFORM_PASSWORD` | Basic auth password |
| `ITENTIAL_MCP_PLATFORM_CLIENT_ID` | OAuth 2.0 client ID |
| `ITENTIAL_MCP_PLATFORM_CLIENT_SECRET` | OAuth 2.0 client secret |

##### imagePullSecrets

If pulling the image from a private registry, create an image pull secret and reference it
with `imagePullSecrets[0].name` in your values file.

### Transport Mode

The MCP server supports three transport modes. Only `sse` and `http` are suitable for
Kubernetes deployments. The default is `sse`.

| Mode | Description |
|:-----|:------------|
| `sse` | Server-Sent Events. Use for web and LLM client integrations. |
| `http` | Stateless HTTP. Use for service mesh deployments. |
| `stdio` | Standard I/O. Local CLI use only — do not use in Kubernetes. |

### TLS and Certificate Verification

The MCP server connects to Itential Platform over HTTPS by default. If IAP is using a
self-signed certificate or an internal CA, set `mcp.platform.disableVerify: true` to skip
certificate verification while keeping the connection encrypted.

```yaml
mcp:
  platform:
    disableVerify: true
```

Do **not** use `mcp.platform.disableTls: true` unless IAP is genuinely running without TLS.
That setting disables TLS entirely and switches the client to HTTP, which will fail if IAP is
listening on an HTTPS port.

### Integrating with Claude Desktop

Claude Desktop's `claude_desktop_config.json` only supports stdio-based MCP servers. To connect
it to this Kubernetes-deployed server you need
[mcp-remote](https://github.com/geelen/mcp-remote), a small local proxy that bridges Claude
Desktop's stdio requirement to the remote HTTP/SSE endpoint.

#### 1. Install mcp-remote

```bash
npm install -g mcp-remote
```

#### 2. Extract the cluster CA certificate

If the ingress uses a certificate signed by an internal CA (e.g. cert-manager with a custom
ClusterIssuer), extract the root CA and save it to a stable path on the machine running Claude
Desktop.

```bash
# Extract the root CA from the TLS secret used by the ingress
kubectl get secret <tls-secret-name> -n <namespace> \
  -o jsonpath='{.data.ca\.crt}' | base64 -d > ~/.config/itential/certs/itential-root-ca.crt
```

#### 3. Configure claude_desktop_config.json

Add the following entry to `~/Library/Application Support/Claude/claude_desktop_config.json`
(macOS) or `%APPDATA%\Claude\claude_desktop_config.json` (Windows). Use the absolute path to
the `node` binary for the version of Node that has `mcp-remote` installed — Claude Desktop does
not inherit shell environment variables like `nvm` aliases.

```json
{
  "mcpServers": {
    "itential-mcp": {
      "command": "/path/to/node",
      "args": [
        "/path/to/mcp-remote",
        "https://<ingress-hostname>/mcp"
      ],
      "env": {
        "NODE_EXTRA_CA_CERTS": "/path/to/itential-root-ca.crt"
      }
    }
  }
}
```

`NODE_EXTRA_CA_CERTS` trusts the internal CA cert for Node.js TLS connections without
disabling certificate verification. If the ingress certificate is signed by a publicly trusted
CA this variable can be omitted.

Quit and relaunch Claude Desktop after editing the config file.

#### Notes

- `mcp-remote` defaults to `http-first` transport strategy: it tries streamable HTTP (POST)
  first and falls back to SSE. Both work with the default `sse` transport of this chart.
- The `url`-only config format (`{"url": "..."}`) is **not** supported by Claude Desktop's
  current config schema. The `command`/`args` form shown above is required.
- Confirm the connection is working by checking the tool picker (hammer icon) in Claude
  Desktop's chat input bar for the server name.

### Integrating with Claude Code

Claude Code manages MCP servers via the `claude mcp add` command, which writes to
`~/.claude.json`. Do **not** add `mcpServers` to `~/.claude/settings.json` — that file is not
read for MCP server registration.

#### 1. Install mcp-remote

Follow the same mcp-remote installation step from the Claude Desktop section above if you
have not done so already.

#### 2. Extract the cluster CA certificate

Follow the same CA certificate extraction step from the Claude Desktop section above if you
have not done so already.

#### 3. Register the server

```bash
claude mcp add itential-mcp-kv \
  -s user \
  -e NODE_EXTRA_CA_CERTS=/path/to/itential-root-ca.crt \
  -- /path/to/node \
     /path/to/mcp-remote \
     https://<ingress-hostname>/mcp
```

Use absolute paths for `node` and `mcp-remote`. To find them:

```bash
# Find the node binary for the version that has mcp-remote installed
which node   # or: nvm which <version>

# Find mcp-remote
which mcp-remote   # or: find ~/.nvm -name mcp-remote
```

The `-s user` flag registers the server in your user-level config so it is available in every
Claude Code session, not just the current project.

#### 4. Verify the connection

```bash
claude mcp list
```

The server should show `✔ Connected`. Start a new Claude Code session to begin using the
tools.

### Tool Filtering

The MCP server exposes 56+ tools by default. You can reduce the exposed surface using the
`mcp.platform.includeTags` and `mcp.platform.excludeTags` values. These accept a
comma-separated list of tool group tags.

```bash
helm install mcp . -f values.yaml \
  --set mcp.platform.includeTags="health,configuration_manager"
```

## Values

| Key | Type | Default | Description |
|:----|:-----|:--------|:------------|
| affinity | object | `{}` | Additional affinities |
| applicationPort | int | `8000` | The port the MCP server container listens on inside the pod. Must match `mcp.server.port`. |
| credentials.secretName | string | `""` | Name of an existing Secret to use for platform credentials. When set, no Secret is created by the chart. |
| credentials.platformUser | string | `""` | Basic auth username for Itential Platform. Only used when `credentials.secretName` is empty. |
| credentials.platformPassword | string | `""` | Basic auth password for Itential Platform. Only used when `credentials.secretName` is empty. |
| credentials.platformClientId | string | `""` | OAuth 2.0 client ID. Only used when `credentials.secretName` is empty. |
| credentials.platformClientSecret | string | `""` | OAuth 2.0 client secret. Only used when `credentials.secretName` is empty. |
| deployment.enabled | bool | `true` | Toggle creation of the Deployment object. |
| image.pullPolicy | string | `"IfNotPresent"` | The image pull policy. |
| image.repository | string | `"ghcr.io/itential/itential-mcp"` | The image repository. |
| image.tag | string | `""` | The image tag. Defaults to the chart `appVersion` when empty. |
| imagePullSecrets | list | `[]` | The secrets object used to pull the image from the repo. |
| ingress.annotations | object | `nil` | Annotations for the Ingress object. |
| ingress.className | string | `""` | The ingress controller class name. |
| ingress.enabled | bool | `false` | Toggle creation of the Ingress object. |
| ingress.hosts | list | See `values.yaml` | List of hosts and paths for the Ingress. |
| ingress.name | string | `"mcp-ingress"` | The name of the Kubernetes Ingress object. |
| ingress.tls | list | `[]` | TLS configuration for the Ingress. |
| livenessProbe.enabled | bool | `false` | Toggle the liveness probe. |
| livenessProbe.failureThreshold | int | `3` | Number of failures before the pod is killed. |
| livenessProbe.path | string | `"/mcp"` | HTTP path for the liveness probe. |
| livenessProbe.periodSeconds | int | `30` | How often to run the probe. |
| livenessProbe.successThreshold | int | `1` | Minimum consecutive successes to be considered healthy. |
| livenessProbe.timeoutSeconds | int | `5` | Seconds after which the probe times out. |
| mcp.platform.disableTls | bool | `false` | Disable TLS entirely for the platform connection (connects via HTTP). Only use if IAP is not running TLS. For internal CA certs, use `disableVerify` instead. |
| mcp.platform.disableVerify | bool | `false` | Skip TLS certificate verification when connecting to Itential Platform. Use when IAP uses a self-signed or internal CA cert. |
| mcp.platform.excludeTags | string | `""` | Comma-separated list of tool tag groups to hide. |
| mcp.platform.host | string | `""` | Hostname or IP address of the Itential Platform instance. |
| mcp.platform.includeTags | string | `""` | Comma-separated list of tool tag groups to expose. Leave empty to expose all. |
| mcp.platform.port | int | `3000` | Port of the Itential Platform instance. Set to `0` for auto-detection. |
| mcp.platform.timeout | int | `30` | Connection timeout in seconds. |
| mcp.server.host | string | `"0.0.0.0"` | The address the MCP server binds to inside the container. |
| mcp.server.path | string | `"/mcp"` | The HTTP path the MCP server mounts on. |
| mcp.server.port | int | `8000` | The port the MCP server listens on. Must match `applicationPort`. |
| mcp.server.transport | string | `"sse"` | Transport mode. Use `sse` or `http` for Kubernetes. |
| nodeSelector | object | `{"itential.io/app":"mcp"}` | Node selector labels. |
| podAnnotations | object | `{}` | Additional pod annotations. |
| podLabels | object | `{}` | Additional pod labels. |
| podSecurityContext | object | `{}` | Pod-level security context. |
| readinessProbe.enabled | bool | `false` | Toggle the readiness probe. |
| readinessProbe.failureThreshold | int | `3` | Number of failures before the pod is marked not ready. |
| readinessProbe.path | string | `"/mcp"` | HTTP path for the readiness probe. |
| readinessProbe.periodSeconds | int | `10` | How often to run the probe. |
| readinessProbe.successThreshold | int | `1` | Minimum consecutive successes to be considered ready. |
| readinessProbe.timeoutSeconds | int | `5` | Seconds after which the probe times out. |
| replicaCount | int | `1` | The number of pods to start. |
| resources.limits.cpu | string | `"500m"` | CPU limit. |
| resources.limits.memory | string | `"256Mi"` | Memory limit. |
| resources.requests.cpu | string | `"250m"` | CPU request. |
| resources.requests.memory | string | `"128Mi"` | Memory request. |
| resourcesEnabled | bool | `true` | Toggle resource limits and requests on the container. |
| securityContext | object | `{}` | Container-level security context. |
| service.name | string | `"mcp-service"` | The name of the Kubernetes Service object. |
| service.port | int | `8000` | The port the Service listens on. |
| service.type | string | `"ClusterIP"` | The Service type. |
| serviceAccount.name | string | `""` | The name of the service account to assign to the pods. When empty, Kubernetes uses the default service account in the namespace. |
| startupProbe.enabled | bool | `false` | Toggle the startup probe. |
| startupProbe.failureThreshold | int | `18` | Number of failures before the pod is killed during startup. |
| startupProbe.initialDelaySeconds | int | `10` | Seconds before the startup probe begins. |
| startupProbe.path | string | `"/mcp"` | HTTP path for the startup probe. |
| startupProbe.periodSeconds | int | `10` | How often to run the probe during startup. |
| startupProbe.successThreshold | int | `1` | Minimum consecutive successes to pass startup. |
| startupProbe.timeoutSeconds | int | `5` | Seconds after which the startup probe times out. |
| tolerations | list | See `values.yaml` | Pod tolerations. |
| volumeMounts | list | `[]` | Additional volume mounts on the Deployment definition. |
| volumes | list | `[]` | Additional volumes on the Deployment definition. |
