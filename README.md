# eks-service-mesh-istio

> Production Istio setup on EKS: mTLS everywhere, traffic management, circuit breaking, and observability wired to Prometheus + Grafana.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

## What this covers

- **mTLS** — automatic mutual TLS between all services (STRICT mode)
- **Traffic management** — VirtualServices, DestinationRules, retries, timeouts
- **Circuit breaking** — outlier detection, connection pool limits
- **Observability** — Kiali topology map, Jaeger traces, Prometheus metrics
- **Ingress** — Istio Gateway replacing ALB for L7 routing
- **Canary** — traffic splitting via VirtualService weights

## Architecture

```
External Traffic
      │
      ▼
┌─────────────────┐
│  Istio Gateway  │  (LoadBalancer Service → NLB)
└────────┬────────┘
         │
         ▼
┌─────────────────────────────────────────────┐
│              Service Mesh                    │
│                                             │
│  [Service A] ──mTLS──► [Service B]          │
│       │                      │              │
│  Envoy Sidecar          Envoy Sidecar       │
│  (metrics/traces)       (metrics/traces)    │
└─────────────────────────────────────────────┘
         │
         ▼
┌────────────────────────────────┐
│  Telemetry Backends            │
│  Prometheus / Grafana / Jaeger │
└────────────────────────────────┘
```

## Install

```bash
# Install Istio via istioctl
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.21.0 sh -
export PATH=$PWD/istio-1.21.0/bin:$PATH

istioctl install -f configs/istio-operator.yaml --verify

# Enable sidecar injection on your namespaces
kubectl label namespace production istio-injection=enabled

# Apply traffic policies
kubectl apply -f configs/
```

## Key configs

### Enforce mTLS cluster-wide
```yaml
# configs/peer-authentication.yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT
```

### Circuit breaker
```yaml
# configs/destination-rule-circuit-breaker.yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: my-service
spec:
  host: my-service
  trafficPolicy:
    connectionPool:
      http:
        http1MaxPendingRequests: 100
        http2MaxRequests: 1000
    outlierDetection:
      consecutive5xxErrors: 3
      interval: 10s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
```

### Canary traffic split
```yaml
# configs/virtual-service-canary.yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: my-service
spec:
  hosts: [my-service]
  http:
    - route:
        - destination:
            host: my-service
            subset: stable
          weight: 90
        - destination:
            host: my-service
            subset: canary
          weight: 10
```

## Files

```
eks-service-mesh-istio/
├── configs/
│   ├── istio-operator.yaml
│   ├── peer-authentication.yaml       # mTLS STRICT
│   ├── destination-rule-circuit-breaker.yaml
│   ├── virtual-service-canary.yaml
│   └── gateway.yaml
├── terraform/
│   └── main.tf                        # Istio via Helm + Terraform
└── runbooks/
    ├── 01-mtls-troubleshooting.md
    └── 02-traffic-management.md
```

## License

MIT — by [Goutham Annem](https://linkedin.com/in/goutham-annem)
