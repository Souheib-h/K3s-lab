# k3s-lab

Cluster K3s HA avec external PostgreSQL — lab d'apprentissage.

## Architecture

| VM | Rôle | IP |
|----|------|----|
| Load-srvs | LB Control Plane | 10.10.0.10 |
| K3s-srv-1 | Server Node 1 | 10.10.0.11 |
| K3s-srv-2 | Server Node 2 | 10.10.0.12 |
| K3s-db | PostgreSQL | 10.10.0.20 |
| Load-agents | LB Workers | 10.10.0.30 |
| K3s-agent-node-1 | Worker 1 | 10.10.0.31 |
| K3s-agent-node-2 | Worker 2 | 10.10.0.32 |
| K3s-agent-node-3 | Worker 3 | 10.10.0.33 |

## Stack

- K3s (Kubernetes léger)
- PostgreSQL (datastore externe)
- HAProxy (load balancer)
- Ubuntu 22.04 (servers/agents/db)
- Alpine (load balancers)
