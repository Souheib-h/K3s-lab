# 04 — Installation des nœuds serveurs K3s

## Architecture

| VM | Rôle | IP |
|---|---|---|
| K3s-srv-1 | Nœud Serveur 1 (control-plane) | 10.10.0.11 |
| K3s-srv-2 | Nœud Serveur 2 (control-plane) | 10.10.0.12 |

Les deux nœuds serveurs partagent le même datastore PostgreSQL (`K3s-db`) — c'est ce qui permet la Haute-Disponibilité. Si un serveur tombe, l'autre continue à gérer le cluster.

---

## Prérequis

Vérifier que PostgreSQL est accessible depuis chaque nœud serveur :

```bash
# Installer le client PostgreSQL
sudo apt update && sudo apt install -y postgresql-client

# Tester la connexion
psql -h 10.10.0.20 -U k3s -d k3s -c "\conninfo"
# Password : k3s_password
```

Autoriser l'accès à la base `postgres` (nécessaire pour K3s au démarrage) :

```bash
# Depuis le host
ssh -t k3s-admin@10.10.0.20 "sudo bash -c 'echo \"host    all     k3s     10.10.0.0/24    md5\" >> /etc/postgresql/16/main/pg_hba.conf && systemctl reload postgresql'"
```

---

## Installation K3s-srv-1

```bash
ssh k3s-admin@10.10.0.11

curl -sfL https://get.k3s.io | sh -s - server \
  --datastore-endpoint="postgres://k3s:k3s_password@10.10.0.20:5432/k3s" \
  --tls-san 10.10.0.10 \
  --tls-san 10.10.0.11
```

### Récupérer le token

```bash
sudo cat /var/lib/rancher/k3s/server/token
```

> Ce token est nécessaire pour joindre srv-2 et tous les workers.

---

## Installation K3s-srv-2

```bash
ssh k3s-admin@10.10.0.12

curl -sfL https://get.k3s.io | K3S_TOKEN="<token-de-srv-1>" sh -s - server \
  --datastore-endpoint="postgres://k3s:k3s_password@10.10.0.20:5432/k3s" \
  --tls-san 10.10.0.10 \
  --tls-san 10.10.0.12
```

> **Important** : spécifier le token explicitement via `K3S_TOKEN` pour que srv-2 rejoigne le même cluster que srv-1. Sans ça, srv-2 génère son propre bootstrap et entre en conflit avec PostgreSQL.

---

## Vérification

```bash
sudo kubectl get nodes
```

```
NAME        STATUS   ROLES           AGE     VERSION
k3s-srv-1   Ready    control-plane   17m     v1.35.5+k3s1
k3s-srv-2   Ready    control-plane   5m40s   v1.35.5+k3s1
```

```bash
sudo kubectl get pods -A
```

![Cluster serveurs Ready](img/04-cluster-serveurs-ready.png)