# Design Document: Kubernetes Replica Management API

## Overview
- **Purpose**: Minimal Go server for K8s cluster interaction. Built for efficiency - every feature pulls double duty, every line of code earns its place. 

- **Scope**: 
  - Implemented
    - HTTP API to:
    - Retrieve and set the replica count of a Kubernetes Deployment
    - List Deployments
    - Health check (Kubernetes connectivity)
    - Deployment replica count is **cached** by watching Deployments; read-only requests do not hit cluster directly
    - **mTLS** secures HTTP API connections
    - Minimal test coverage (happy/unhappy paths)
    - Dockerfile for server build
    - Makefile for automation/testing/deployment
    - Helm chart including Deployment, ServiceAccount, and Service
    - Local deployment using KIND (macOS/Linux compatible)
    - Deployment supports zero-downtime upgrades
  - Deliberate Exclusions (with TODOs)
    - **No multi-cluster management**
        - TODO: Extend service to support multiple Kubernetes clusters.
    - **No production-grade configuration management**
        - TODO: Switch to environment/config-based settings for all service variables.
    - **No advanced error reporting or structured observability/logging**
        - TODO: Integrate structured logging, metrics, tracing, and advanced failure reporting.
    - **No rate limiting or DoS protection**
        - TODO: Add rate limiting for public APIs.
    - **No integration with external secrets management or RBAC auditing**
        - TODO: Add external secret support and detailed RBAC policy/auditing.
    - **No CI/CD pipeline**
        - TODO: Implement full CI/CD workflow for builds/tests/deployments.

- **Key Trade-offs**: 
  - **Performance over cluster load**: Caching with watchers reduces API calls but adds memory overhead
  - **Security over simplicity**: mTLS authentication increases deployment complexity  
  - **Development speed over production features**: Comprehensive local workflow, minimal observability
  - **Simplicity over enterprise scale**: Single-cluster design limits multi-tenancy

## API Design
### REST Endpoints
- `GET /deployments`: List available Deployments
- `GET /deployment/{name}/replicas`: Get replica count (cached)
- `POST /deployment/{name}/replicas`: Set replica count
- `GET /healthz`: Health check for Kubernetes connectivity
- **Request/response schemas**: JSON with replica counts, timestamps, validation errors
- **Error handling**: Standard HTTP codes (200, 400, 404, 500) with structured error responses

## Architecture & Workflow
### System Architecture
- **HTTP Server**: Go stdlib with custom mTLS handler and JSON routing
- **Kubernetes Client**: client-go SharedInformerFactory for watch-based caching  
- **Security Layer**: Self-signed CA with mounted certificates via Kubernetes secrets

### Caching Strategy
- **Implementation**: client-go SharedInformerFactory with watch-based updates
- **Performance**: O(1) reads from in-memory cache, zero cluster queries for GET requests
- **Consistency**: Real-time cache updates via Kubernetes watch events (no polling)
- **Fallback**: Direct API query on cache miss with immediate cache population

### mTLS Security
- **Certificates**: Self-signed CA with 4096-bit RSA keys (TODO: cert-manager integration)
- **TLS Config**: TLS 1.3 minimum, AEAD ciphers only, RequireAndVerifyClientCert
- **Client Auth**: Certificate-based identity with CN-based role mapping
- **Deployment**: Certificates mounted as Kubernetes secrets with proper RBAC

## Developer Workflow
**Prerequisites**: Go 1.21+, Docker, kubectl, Helm 3.x, KIND, OpenSSL

**Complete Setup** (from clean checkout):
```bash
make kind-up              # Create local K8s cluster  
make cert-generate        # Generate mTLS certificates
make build test           # Build binary and run unit tests
make docker-build kind-load # Build and load container image
make helm-install         # Deploy service with zero-downtime config
make integration-test     # Full end-to-end test suite
```

**Key Automation**:
- **Testing**: Unit tests (table-driven), integration tests (real KIND cluster)
- **Deployment**: Rolling updates with readiness probes, zero-downtime upgrades
- **Security**: Automated certificate generation and Kubernetes secret provisioning

## Build, Test, & Release Automation
- **Build**: Go modules, reproducible builds
- **Testing**:
  - Unit tests (table-driven, key logic)
  - Integration tests (KIND cluster; happy/unhappy paths)
- **CI/CD**: Local automation mimics production pipeline (TODO: GitHub Actions integration)

## Helm Chart & Deployment
- **Chart Components**: Deployment (with resource limits), ServiceAccount (minimal RBAC), Service (ClusterIP)
- **Zero-Downtime Strategy**: Rolling updates with `maxUnavailable: 0`, readiness probes on `/healthz`
- **Certificate Management**: CA and TLS certs deployed as Kubernetes secrets with proper file modes

## Production Readiness Assessment

**Production Ready Features**:
- **High Availability**: Zero-downtime rolling updates, readiness/liveness probes
- **Performance**: Cached reads (<10ms), event-driven updates, minimal cluster load  
- **Security**: mTLS authentication, certificate-based client identity
- **Automation**: Complete build/test/deploy pipeline, local K8s development
- **Quality**: Unit + integration tests, error handling, graceful shutdown

**Known Production Gaps**:
- **Observability**: No metrics/logging/tracing (Prometheus/Jaeger planned)
- **Configuration**: Hardcoded values (ConfigMap/environment migration planned)
- **Scale**: Single-cluster support (multi-cluster federation planned)
- **Resilience**: No rate limiting/DoS protection (admission webhooks planned)

## Trade-Offs & Deferred Items

**Design Philosophy**: Minimal code, maximum impact. Single binary handles HTTP serving, caching, certificate management, and K8s integration in <2000 lines.

**Key Decisions**:
- **Caching over simplicity**: Adds ~100 lines but eliminates cluster load for reads
- **Self-signed certs over external PKI**: Reduces dependencies, increases setup complexity  
- **Single binary over microservices**: Simpler deployment, harder to scale individual components
- **Hardcoded config over external config**: Faster development, requires rebuilds for changes

**Deferred for complexity/time**:
- Multi-cluster support (federation complexity)
- Advanced observability (Prometheus/Jaeger integration)  
- External secret management (Vault/AWS integration)
- Comprehensive RBAC auditing (logging/monitoring overhead)

## References

**Kubernetes Documentation**:
- [client-go](https://github.com/kubernetes/client-go) - Official Go client library
- [SharedInformers](https://pkg.go.dev/k8s.io/client-go/informers) - Efficient resource watching
- [Rolling Updates](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#rolling-update-deployment) - Zero-downtime deployments

**Security & Best Practices**:
- [TLS 1.3 RFC](https://tools.ietf.org/html/rfc8446) - Modern TLS implementation  
- [Helm Best Practices](https://helm.sh/docs/chart_best_practices/) - Production Helm charts
- [KIND](https://kind.sigs.k8s.io/) - Local Kubernetes development

**Challenge Reference**:
- [Teleport SRE Challenge](https://github.com/gravitational/careers/blob/main/challenges/sre/challenge.md)