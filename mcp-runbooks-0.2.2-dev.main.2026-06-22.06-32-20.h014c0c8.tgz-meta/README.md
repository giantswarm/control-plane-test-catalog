# mcp-runbooks

MCP (Model Context Protocol) server for serving Giant Swarm operational runbooks.

## Overview

This server provides AI assistants with access to Giant Swarm runbooks through the MCP protocol. It supports searching, listing, and reading runbooks with full frontmatter metadata extraction.

## Available Tools

| Tool | Description | Parameters |
|------|-------------|------------|
| `search_runbooks` | Full-text search across all runbooks | `query: string` (required) |
| `list_runbooks` | List all runbooks with metadata | `category?: string` (optional filter) |
| `read_runbook` | Read full content of a runbook | `name: string` (required) |
| `get_runbook_metadata` | Get frontmatter metadata | `name: string` (required) |

## Local Development

### Prerequisites

- Go 1.22+
- Make

### Build

```bash
make build
```

### Run locally

```bash
make run
```

Or with custom runbooks path:

```bash
RUNBOOKS_PATH=/path/to/runbooks ./bin/mcp-runbooks
```

### Run tests

```bash
make test
```

## Transport Modes

The server supports three transport modes:

| Transport | Flag | Use Case | Endpoint |
|-----------|------|----------|----------|
| `stdio` | `-transport=stdio` | Local/desktop AI assistants | stdin/stdout |
| `sse` | `-transport=sse` | Legacy HTTP clients | `/sse`, `/message` |
| `http` | `-transport=http` | **Recommended** — Streamable HTTP (MCP standard) | `/mcp` |

All HTTP transports expose health endpoints at `/health`, `/healthz`, `/ready`, `/readyz`.

## Kubernetes Deployment

### Install with Helm

Using image-embedded runbooks (recommended):

```bash
helm install mcp-runbooks ./helm/mcp-runbooks \
  --namespace mcp-runbooks --create-namespace \
  --set mcp.transport=http \
  --set runbooks.useConfigMap=false \
  --set runbooks.path=/app/runbooks \
  --set image.repository=gsociprivate.azurecr.io/giantswarm/mcp-runbooks
```

Using a ConfigMap for runbooks (for custom runbook content):

```bash
helm install mcp-runbooks ./helm/mcp-runbooks \
  --namespace mcp-runbooks --create-namespace \
  --set mcp.transport=http
```

### Upgrade

```bash
helm upgrade mcp-runbooks ./helm/mcp-runbooks \
  --namespace mcp-runbooks \
  --set mcp.transport=http \
  --set runbooks.useConfigMap=false \
  --set runbooks.path=/app/runbooks \
  --set image.repository=gsociprivate.azurecr.io/giantswarm/mcp-runbooks
```

### Configuration

See [helm/mcp-runbooks/values.yaml](helm/mcp-runbooks/values.yaml) for all available options.

Key configuration:

```yaml
mcp:
  # Transport: sse or http (streamable HTTP, recommended)
  transport: http
  addr: ":8080"

runbooks:
  # Path inside the container where runbooks are located
  path: /app/runbooks
  # Set to true to mount a ConfigMap instead of using image-embedded runbooks
  useConfigMap: false

image:
  repository: gsociprivate.azurecr.io/giantswarm/mcp-runbooks
  tag: "0.1.0"

service:
  type: ClusterIP
  port: 8080

monitoring:
  enabled: true
  serviceMonitor:
    enabled: true
```

### Verify Deployment

```bash
kubectl port-forward -n mcp-runbooks svc/mcp-runbooks 8080:8080

# Health check
curl http://localhost:8080/health

# MCP initialize (streamable HTTP)
curl -X POST http://localhost:8080/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-03-26","capabilities":{},"clientInfo":{"name":"test","version":"1.0.0"}}}'
```

## MCP Client Configuration

### stdio transport (local)

```json
{
  "mcpServers": {
    "runbooks": {
      "command": "/path/to/mcp-runbooks",
      "env": {
        "RUNBOOKS_PATH": "/path/to/runbooks"
      }
    }
  }
}
```

### Streamable HTTP transport (remote)

```json
{
  "mcpServers": {
    "runbooks": {
      "url": "http://mcp-runbooks.mcp-runbooks.svc.cluster.local:8080/mcp"
    }
  }
}
```

## License

Apache 2.0

## Contributing

Please read [CONTRIBUTING.md](CONTRIBUTING.md) for details on our code of conduct and the process for submitting pull requests.
