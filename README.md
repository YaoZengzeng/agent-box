# Pico API Server

A lightweight API server for managing Kubernetes Sandbox environments with transparent SSH/SFTP proxy support.

## Features

- **Session Management**: RESTful API for creating, listing, and deleting sandbox sessions
- **Kubernetes Integration**: Automatically manages Sandbox CRD lifecycle
- **HTTP CONNECT Tunnel**: Transparent proxy for SSH/SFTP traffic to sandbox pods
- **Authentication**: JWT Bearer token authentication support
- **TLS Support**: Optional HTTPS for secure communication

## Quick Start

### Option 1: Deploy to Kubernetes (Recommended)

```bash
# Build Docker image
make docker-build

# For kind cluster
make kind-load

# Deploy to Kubernetes
make k8s-deploy

# Check status
kubectl get pods -n pico-apiserver
make k8s-logs

# Port forward for testing
kubectl port-forward -n pico-apiserver svc/pico-apiserver 8080:8080
```

See [k8s/README.md](k8s/README.md) for detailed deployment instructions.

### Option 2: Local Development

```bash
# Build binary
make build

# Run locally (no Kubernetes required for testing)
./bin/pico-apiserver --port=8080

# Or with Kubernetes
./bin/pico-apiserver \
  --port=8080 \
  --kubeconfig=$HOME/.kube/config \
  --namespace=sandboxes
```

### Test

```bash
# Health check
curl http://localhost:8080/health

# Create session (requires Kubernetes)
curl -X POST http://localhost:8080/v1/sessions \
  -H "Authorization: Bearer token" \
  -H "Content-Type: application/json" \
  -d '{"ttl": 3600, "image": "python:3.11"}'
```

## Architecture

```
Client → REST API → pico-apiserver → Kubernetes API → Sandbox CRD
                                                             ↓
                                                  agent-sandbox controller
                                                             ↓
                                                         Pod created
```

### HTTP CONNECT Tunnel

```
Client <--HTTP CONNECT--> pico-apiserver <--TCP/SSH--> Sandbox Pod
```

The server acts as a transparent proxy:
1. Client sends `CONNECT /v1/sessions/{id}/tunnel HTTP/1.1`
2. Server connects to sandbox pod SSH service
3. Server hijacks HTTP connection
4. Server returns `200 Connection Established`
5. Bidirectional transparent data forwarding begins

## API Endpoints

### Sessions

- `POST /v1/sessions` - Create session
- `GET /v1/sessions` - List sessions
- `GET /v1/sessions/{id}` - Get session details
- `DELETE /v1/sessions/{id}` - Delete session

### Tunnel

- `CONNECT /v1/sessions/{id}/tunnel` - Establish SSH/SFTP tunnel

### Health

- `GET /health` - Health check (no auth required)

## Python SDK Usage

```python
from sdk.sandbox_sessions_sdk import SessionsClient, SessionSSHClient

# Create session
with SessionsClient(api_url='http://localhost:8080/v1', bearer_token='token') as client:
    session = client.create_session(ttl=3600)
    print(f"Session: {session.session_id}")

# Use SSH through tunnel
with SessionSSHClient(
    api_url='http://localhost:8080/v1',
    bearer_token='token',
    session_id=session.session_id,
    username='sandbox'
) as ssh:
    result = ssh.run_command('ls -la')
    print(result['stdout'])
```

## Configuration

### Command Line Options

| Flag             | Default   | Description                             |
| ---------------- | --------- | --------------------------------------- |
| `--port`         | `8080`    | API server port                         |
| `--kubeconfig`   | `""`      | Path to kubeconfig (empty = in-cluster) |
| `--namespace`    | `default` | Kubernetes namespace                    |
| `--ssh-username` | `sandbox` | SSH username for pods                   |
| `--ssh-port`     | `22`      | SSH port on pods                        |
| `--enable-tls`   | `false`   | Enable HTTPS                            |
| `--tls-cert`     | `""`      | TLS certificate file                    |
| `--tls-key`      | `""`      | TLS key file                            |
| `--jwt-secret`   | `""`      | JWT secret (empty = skip auth)          |

## Project Structure

```
agent-box/
├── cmd/pico-apiserver/          # Entry point
├── pkg/pico-apiserver/          # Core implementation
│   ├── apiserver.go            # Server, routing, middleware
│   ├── tunnel.go               # ⭐ HTTP CONNECT tunnel & proxy
│   ├── handlers.go             # REST API handlers
│   ├── k8s_client.go           # Kubernetes client
│   ├── session.go              # Session management
│   ├── auth.go                 # Authentication
│   ├── config.go               # Configuration
│   └── utils.go                # Utilities
├── sdk/                         # Python SDK (reference)
├── api-spec/                    # OpenAPI specification
└── Makefile                     # Build system
```

## Development Status

### ✅ Completed (Framework)

- HTTP server and routing
- Session management API
- HTTP CONNECT tunnel handling
- Transparent TCP proxy
- Kubernetes client wrapper
- Authentication middleware
- Build system

### 🚧 TODO (Requires Debugging)

- Kubernetes Sandbox CRD integration (adjust to actual spec)
- Pod IP retrieval and labeling
- Session TTL cleanup
- JWT authentication implementation
- Pod readiness waiting
- Error handling improvements
- Unit tests

## Dependencies

- `github.com/gorilla/mux` - HTTP routing
- `github.com/google/uuid` - UUID generation
- `golang.org/x/crypto/ssh` - SSH client
- `k8s.io/client-go` - Kubernetes client
- `k8s.io/apimachinery` - Kubernetes API machinery

## References

- API Spec: `api-spec/sandbox-api-spec.yaml`
- Python SDK: `sdk/sandbox_sessions_sdk.py`
- Agent Sandbox: https://github.com/kubernetes-sigs/agent-sandbox

## License

Apache 2.0

## Contributing

Contributions welcome! This is a framework implementation that requires:
1. Integration with actual agent-sandbox CRD
2. Production-ready authentication
3. Comprehensive testing
4. Performance optimization

## Notes

- Current implementation uses placeholder Sandbox CRD structure
- JWT validation needs proper implementation
- Production deployments should verify SSH host keys
- Session cleanup requires background goroutine implementation

