# k3s-lab

> Cluster K3s en mode Haute-Disponibilité avec datastore PostgreSQL externe.  
> Lab d'apprentissage Kubernetes sur VMs KVM locales.

---

## Architecture
```mermaid
graph LR
    kubectl -->|port 6443| LB_SRV[ Load-srvs: HAProxy 10.10.0.10]
    LB_SRV --> SRV1[K3s-srv-1 : Nœud Serveur 10.10.0.11]
    LB_SRV --> SRV2[K3s-srv-2 : Nœud Serveur 10.10.0.12]
    LB_SRV --> SRV3[K3s-srv-3 : Nœud Serveur 10.10.0.13]
    SRV1 --> DB[K3s-db : PostgreSQL 10.10.0.20]
    SRV2 --> DB
    SRV3 --> DB

    apps -->|trafic 80/443| TRAEFIK[Traefik + ServiceLB\nsur chaque nœud]
    TRAEFIK --> AGT1[K3s-agent-node-1: Nœud Worker 10.10.0.31]
    TRAEFIK --> AGT2[K3s-agent-node-2: Nœud Worker 10.10.0.32]
    TRAEFIK --> AGT3[K3s-agent-node-3: Nœud Worker 10.10.0.33]
```

> `Load-agents` (10.10.0.30) a été retiré à l'étape 7 : Traefik + ServiceLB (klipper-lb) exposent déjà les ports 80/443 sur chaque nœud.

---

![Diagramme par défaut de Rancher](docs/img/Default-diagrame.png)

## Plan d'adressage

| VM                 | Rôle                          | IP           | OS           |
| ------------------ | ----------------------------- | ------------ | ------------ |
| `Load-srvs`        | Load Balancer — Control Plane | `10.10.0.10` | Alpine       |
| `K3s-srv-1`        | Nœud Serveur 1                | `10.10.0.11` | Ubuntu 24.04 |
| `K3s-srv-2`        | Nœud Serveur 2                | `10.10.0.12` | Ubuntu 24.04 |
| `K3s-srv-3`        | Nœud Serveur 3                | `10.10.0.13` | Ubuntu 26.04 |
| `K3s-db`           | Datastore PostgreSQL          | `10.10.0.20` | Ubuntu 24.04 |
| ~~`Load-agents`~~  | ~~Load Balancer — Workers~~ (retiré, étape 7) | `10.10.0.30` | Alpine |
| `K3s-agent-node-1` | Nœud Worker 1                 | `10.10.0.31` | Ubuntu 24.04 |
| `K3s-agent-node-2` | Nœud Worker 2                 | `10.10.0.32` | Ubuntu 24.04 |
| `K3s-agent-node-3` | Nœud Worker 3                 | `10.10.0.33` | Ubuntu 24.04 |

---

## Stack technique

| Composant        | Rôle                                             |
| ---------------- | ------------------------------------------------ |
| **K3s**          | Distribution Kubernetes légère (Rancher)         |
| **PostgreSQL**   | Datastore externe pour la HA                     |
| **HAProxy**      | Load balancer du control plane (API 6443)        |
| **Ubuntu 24.04** | OS des nœuds serveurs, agents et base de données |
| **Alpine Linux** | OS des load balancers                            |

---

## Documentation

| Étape | Description                 | Lien                                                       | Status |
| ----- | --------------------------- | ---------------------------------------------------------- | ------ |
| 1     | Réseau & plan d'adressage   | [docs/01-reseau.md](docs/01-reseau.md)                     | ✅      |
| 2     | Installation PostgreSQL     | [docs/02-base-de-donnees.md](docs/02-base-de-donnees.md)   | ✅      |
| 3     | Configuration HAProxy       | [docs/03-load-balancer.md](docs/03-load-balancer.md)       | ✅      |
| 4     | Installation nœuds serveurs | [docs/04-cluster-serveurs.md](docs/04-cluster-serveurs.md) | ✅      |
| 5     | Installation nœuds agents   | [docs/05-cluster-agents.md](docs/05-cluster-agents.md)     | ✅      |
| 6     | Premier déploiement test    | [docs/06-deploiement-test.md](docs/06-deploiement-test.md) | ✅      |
| 7     | Ajout K3s-srv-3, retrait Load-agents | [docs/07-ajout-srv3.md](docs/07-ajout-srv3.md) | ✅      |

---

## Réseau

- **Bridge** : `virbr4`
- **Sous-réseau** : `10.10.0.0/24`
- **Passerelle** : `10.10.0.1`
- **Mode** : NAT (accès internet via le host)

