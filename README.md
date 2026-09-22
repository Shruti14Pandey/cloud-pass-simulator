Multi-Tenant Cloud Infrastructure Simulator

A lightweight PaaS (Platform-as-a-Service) control plane built to simulate how cloud platforms isolate tenant workloads, schedule asynchronous infrastructure tasks, and maintain zero-downtime traffic routing during node outages.

Why I Built This

When building multi-tenant developer platforms, two core challenges always come up:

API responsiveness: Provisioning cloud resources takes time. Blocking a REST API thread while Kubernetes spins up namespaces and pods creates poor UX and causes timeouts.

High availability under chaos: If an underlying node dies or a pod gets rescheduled, external load balancers need to route around the failure before dropping user requests.

I built this project to model a working solution to both problems using FastAPI, Redis, Kubernetes, and Azure infrastructure primitives.

Architecture Overview

                      +----------------------------------+
                      |         Client / HTTP Request    |
                      +----------------------------------+
                                        │
                         1. POST /api/v1/deployments
                                        ▼
                      +----------------------------------+
                      |      FastAPI Control Plane       |
                      +----------------------------------+
                                        │
                         2. Enqueue Job (Return 202 Accepted)
                                        ▼
                      +----------------------------------+
                      |       Redis + Celery Queue       |
                      +----------------------------------+
                                        │
                         3. Consume task & execute K8s SDK
                                        ▼
                      +----------------------------------+
                      |   Kubernetes Cluster (AKS/Kind)  |
                      |                                  |
                      |  [ Namespace: tenant-alpha ]     |
                      |   ├── Deployments & Pods          |
                      |   ├── ResourceQuota (CPU/RAM)    |
                      |   └── NetworkPolicy              |
                      +----------------------------------+
                                        ▲
                                        │ 4. Route traffic
                      +----------------------------------+
                      | Azure Load Balancer / Health Probe|
                      +----------------------------------+


Core Workflow

Asynchronous Task Dispatch: The FastAPI control plane accepts deployment specs, assigns a tracking ID, and enqueues the job into Redis/Celery before immediately returning 202 Accepted.

Dynamic Provisioning: Celery workers use the official Python Kubernetes SDK to create an isolated Namespace for the tenant, attach a default NetworkPolicy (denying cross-tenant traffic), and enforce a ResourceQuota.

Fail-Open Load Balancing: The workload exposes health endpoints (/healthz). Azure Load Balancer probes interact with Kubernetes readiness checks to remove failing pods from active backends prior to node teardown.

Tech Stack

Backend & Control Plane: Python 3.11, FastAPI, Celery, Redis

Orchestration & Drivers: Kubernetes Python SDK, Docker, Docker Compose

Infrastructure as Code: Terraform, Azure Kubernetes Service (AKS), Azure Load Balancer

Chaos Testing: Asyncio, HTTPX, Locust

Repository Structure

.
├── control-plane/          # FastAPI application & Celery workers
│   ├── app/
│   │   ├── main.py         # REST API endpoints
│   │   ├── tasks.py        # Background orchestration tasks
│   │   └── k8s_driver.py   # Kubernetes Python SDK implementation
│   └── Dockerfile
├── terraform/              # Azure AKS & Load Balancer IaC scripts
├── chaos-testing/          # Failover traffic & outage simulator scripts
├── k8s/                    # Base network policy & ingress manifests
├── docker-compose.yml      # Local development environment setup
└── README.md


Running Locally

Prerequisites

Docker & Docker Compose

Python 3.11+

kubectl (optional, for inspecting local cluster states)

1. Spin up Control Plane Services

Clone the repo and start the local API server, Redis instance, and Celery worker:

git clone https://github.com/Shruti14Pandey/cloud-pass-simulator.git
cd cloud-pass-simulator
docker-compose up --build -d


Verify the control plane API is live:

curl http://localhost:8000/api/v1/health


2. Trigger a Tenant Deployment

Dispatch an asynchronous deployment task for a tenant:

curl -X POST http://localhost:8000/api/v1/deployments \
  -H "Content-Type: application/json" \
  -d '{
    "tenant_name": "acme-corp",
    "app_image": "nginx:alpine",
    "replicas": 3
  }'


3. Run the Chaos / Failover Test

To test high availability and measure dropped requests during a node failure simulation:

python chaos-testing/simulate_outage.py


Chaos Engineering Test Results

During testing, continuous HTTP GET requests were dispatched against the application endpoint while deleting pods and cordoning nodes in the cluster.

Total Requests Sent: 10,000

Successful (200 OK): 10,000

Failed / Dropped (5xx / Timeout): 0

Success Rate: 100%

Failover Transition Time: ~180ms (Readiness probe rerouted incoming traffic before Kubernetes finalized pod termination).

Future Improvements

Add HashiCorp Vault integration for managing per-tenant secrets dynamically.

Implement Prometheus metrics collection for tracking deployment scheduling latencies.
