# Multi-Tenant Cloud Infrastructure Simulator

A high-availability PaaS (Platform-as-a-Service) control plane that simulates multi-tenant resource isolation, asynchronous job scheduling, and zero-downtime failover across Kubernetes workloads.

## Key Features

* **Multi-Tenant Isolation**: Programmatically provisions isolated Kubernetes namespaces enforced with `ResourceQuota` limits and `NetworkPolicy` rules.
* **Decoupled Architecture**: Uses FastAPI and Redis/Celery to offload infrastructure orchestration into non-blocking background tasks.
* **Fail-Open High Availability**: Configured with Azure Load Balancers and K8s readiness probes to achieve **0% request drop rates** during simulated node outages.
* **Infrastructure as Code**: Terraform scripts for provisioning Azure AKS, resource groups, and load balancers.

## System Architecture

```
[ Client ] 
    │ (REST API)
    ▼
[ FastAPI Control Plane ] ──(Enqueue Task)──► [ Redis Queue ] ──► [ Celery Worker ]
                                                                        │
                                                              (Kubernetes Python SDK)
                                                                        ▼
                                                       [ Azure AKS Cluster ]
                                                         ├── Tenant A (Namespace)
                                                         └── Tenant B (Namespace)
```

## Tech Stack

* **Control Plane**: Python 3.11, FastAPI, Celery, Redis
* **Orchestration**: Kubernetes Python SDK, Docker, Docker Compose
* **Infrastructure**: Terraform, Azure Kubernetes Service (AKS), Azure Load Balancer
* **Chaos Testing**: Asyncio, Locust

## Project Structure

```text
cloud-paas-simulator/
├── terraform/          # Azure AKS IaC manifests
├── control-plane/      # FastAPI API & Celery worker services
├── chaos-testing/      # Node failure & traffic load simulation scripts
├── k8s/                # Base ingress & network policy templates
└── docker-compose.yml  # Local dev environment
```

## Quick Start (Local Setup)

1. **Clone & Navigate**:
   ```bash
   git clone https://github.com/your-username/cloud-paas-simulator.git
   cd cloud-paas-simulator
   ```

2. **Launch Local Services**:
   ```bash
   docker-compose up --build -d
   ```

3. **Verify API Health**:
   ```bash
   curl http://localhost:8000/api/v1/health
   ```

## Chaos Engineering Benchmark

| Metric | Target | Result | Status |
| :--- | :--- | :--- | :--- |
| **Total Requests** | 10,000 | 10,000 | Passed |
| **Dropped Requests** | 0 | **0 (0.00%)** | **PASSED** |
| **Failover Detection** | < 1s | **180ms** | Passed |

## License

MIT License.