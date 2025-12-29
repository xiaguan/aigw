# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

AIGW is an intelligent inference scheduler for large-scale LLM inference services. It provides intelligent routing, overload protection, and multi-tenant QoS capabilities through a global routing solution that is aware of load, KVCache, and LoRA. Built as an Envoy Golang extension integrated with Istio for control plane.

## Build System

The project uses a Makefile-based build system with Docker for reproducible builds.

### Key Build Commands

```bash
# Build the shared object (main artifact)
make build-so              # Build in Docker (recommended)
make build-so-local        # Build locally (requires matching glibc 2.31)

# Testing
make unit-test             # Run unit tests in Docker
make unit-test-local       # Run unit tests locally
make integration-test      # Run integration tests (requires build-test-so first)
make build-test-so         # Build test version with coverage

# Code Quality
make lint-go               # Run golangci-lint (requires golangci-lint v1.62.2)
make lint-license          # Check license headers
make fix-license           # Fix license headers
```

### Development Workflow

```bash
# 1. Build the shared object
make build-so

# 2. Start required services
# First start metadata center (see github.com/aigw-project/metadata-center)

# 3. Start AIGW in local mode (easiest for development)
make start-aigw-local      # Uses etc/envoy-local.yaml + etc/clusters.json

# 4. Test the service
curl localhost:10000/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model": "qwen3", "messages": [{"role": "user", "content": "test"}], "stream": false}'

# 5. Stop when done
make stop-aigw

# Alternative: Start with Istio control plane (for production-like setup)
make start-istio           # Start local Istio
make start-aigw-xds        # Start AIGW with Istio integration
```

### Environment Variables

The Metadata Center connection is configured via environment variables (set in Makefile targets):
- `AIGW_META_DATA_CENTER_HOST` - Defaults to local non-loopback IP
- `AIGW_META_DATA_CENTER_PORT` - Defaults to 8080

Additional tuning variables are documented in `pkg/metadata_center/metadata_center.go:49-60`.

## Architecture

### Core Components

1. **cmd/libgolang** - Entry point that builds the Envoy Golang shared library
2. **plugins/** - Envoy HTTP filters
   - `llmproxy` - Main LLM proxy filter with request transcoding (OpenAI API), load balancing, and cache awareness
3. **pkg/aigateway/** - Core gateway functionality
   - `clustermanager` - Manages clusters and service instances
   - `discovery` - Service discovery (static config and Istio integration)
   - `loadbalancer` - Intelligent load balancing with multi-factor decision algorithm
     - `inferencelb` - The intelligent LB that considers cache hit ratio, request load, and prefill load
4. **pkg/metadata_center/** - Near real-time load metric collection and KV cache tracking
5. **pkg/prediction/** - Latency prediction using Recursive Least Squares (RLS) for TPOT/TTFT

### Data Flow

Request → Envoy → llmproxy filter → transcoder (decode request) → load balancer (select backend) → upstream → transcoder (encode response) → client

The load balancer uses a composite scoring algorithm that weighs:
- Cache hit ratio (weight: 2)
- Request queue load (weight: 1)
- Prefill load (weight: 3)

These weights can be configured per-model via the LB mapping rules.

### Configuration

Two deployment modes:

1. **Local/Static Mode** (`etc/envoy-local.yaml`)
   - Uses `etc/clusters.json` for static service discovery
   - Simple JSON format defining cluster endpoints
   - Best for local development

2. **Istio Mode** (`etc/envoy-istio.yaml`)
   - Uses Istio xDS for dynamic service discovery
   - Reads CRDs from `etc/config_crds/` (ServiceEntry, EnvoyFilter, etc.)
   - Production mode with k8s integration

Model routing and LB behavior are configured through the llmproxy plugin configuration (protobuf-defined in `plugins/llmproxy/config/config.proto`).

## Testing

### Unit Tests

Run with `make unit-test`. Tests use the `envoy${ENVOY_API_VERSION}` build tag (defaults to `envoydev`).

Individual test files:
```bash
go test -tags envoydev -v ./pkg/prediction/... -run TestTPOTPrediction
```

### Integration Tests

Integration tests require building a special test binary with coverage:
```bash
make build-test-so         # Creates tests/integration/libgolang.so
make integration-test      # Runs tests with real Envoy
```

Uses the `integrationtest` build tag. Tests spawn Envoy with the test .so file.

## Code Conventions

- Build tags: Always use `envoy${ENVOY_API_VERSION}` for compatibility with internal Envoy Golang filter version
- Error handling: Use typed errors from `pkg/errcode` for API responses
- Logging: Use `api.LogInfof/LogDebugf` from HTNN API, or `pkg/async_log` for high-volume request logging
- Proto generation: Run `make gen-proto` after modifying .proto files (requires dev-tools image)
- License headers: All files must have Apache 2.0 headers (enforced by `make lint-license`)

## Key Dependencies

- **Envoy v1.35.3** - Proxy runtime
- **Istio v1.27.3** - Control plane (pilot, proxyv2 image)
- **mosn.io/htnn/api** - Envoy Golang filter framework
- **github.com/bytedance/sonic** - Fast JSON encoding/decoding
- **github.com/openai/openai-go** - OpenAI API client (forked version at github.com/aigw-project/openai-go)

## Common Issues

1. **GLIBC mismatch**: The build image (golang:1.22-bullseye) and proxy image (istio/proxyv2:1.27.3) both use glibc 2.31. Building locally may create incompatible binaries.

2. **Static cluster config**: When modifying `etc/clusters.json`, restart AIGW for changes to take effect. The file is mounted into the container.

3. **Metadata center required**: Both local and Istio modes require a running metadata center. Without it, load-aware and cache-aware routing will not function.

4. **Port conflicts**: Local mode uses ports 10000 (AIGW), 10001 (mock backend), 15000 (admin). Istio mode also uses 15010 (xDS), 8080 (pilot).
