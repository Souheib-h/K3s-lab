# 07 — Ajout K3s-srv-3 (3ème control-plane)

## Contexte

Deux décisions d'architecture prises lors de cette étape :

1. **Suppression de Load-agents** — redondant avec klipper-lb/Traefik natif de K3s. Traefik expose déjà automatiquement les services sur l'IP de chaque nœud worker.

2. **Ajout de K3s-srv-3** — prépare la migration vers etcd embarqué. etcd nécessite un nombre impair de nœuds (minimum 3) pour le quorum.

---

## Suppression de Load-agents

```bash
sudo virsh shutdown Load-agents
```

Vérification que Traefik gère l'exposition nativement :

```bash
kubectl get svc -n kube-system traefik
```

```
NAME      TYPE           CLUSTER-IP     EXTERNAL-IP                                              PORT(S)
traefik   LoadBalancer   10.43.118.29   10.10.0.11,10.10.0.12,10.10.0.31,10.10.0.32,10.10.0.33   80:32712/TCP,443:30852/TCP
```

Traefik expose déjà sur les 5 IPs — Load-agents est inutile.

---

## Création de K3s-srv-3

### Clone depuis Base-ubuntu-srv

```bash
sudo virt-clone --original Base-ubuntu-srv --name K3s-srv-3 --file /var/lib/libvirt/images/K3s-srv-3.qcow2
```

### Configuration réseau

```bash
# Récupérer la MAC
sudo virsh domiflist K3s-srv-3

# Changer le réseau default → k3s-net
sudo virsh edit K3s-srv-3
# <source network='k3s-net'/>

# Réserver l'IP 10.10.0.13
sudo virsh net-update k3s-net add ip-dhcp-host \
  "<host mac='52:54:00:4f:39:f8' name='K3s-srv-3' ip='10.10.0.13'/>" \
  --live --config
```

### Plan d'adressage mis à jour

| VM | Rôle | IP | MAC | OS |
|---|---|---|---|---|
| `Load-srvs` | Load Balancer — Control Plane | `10.10.0.10` | `52:54:00:a5:79:bf` | Alpine |
| `K3s-srv-1` | Nœud Serveur 1 | `10.10.0.11` | `52:54:00:89:5a:43` | Ubuntu 24.04 |
| `K3s-srv-2` | Nœud Serveur 2 | `10.10.0.12` | `52:54:00:e2:34:df` | Ubuntu 24.04 |
| `K3s-srv-3` | Nœud Serveur 3 | `10.10.0.13` | `52:54:00:4f:39:f8` | Ubuntu 26.04 |
| `K3s-db` | Datastore PostgreSQL | `10.10.0.20` | `52:54:00:98:ad:7e` | Ubuntu 24.04 |
| `K3s-agent-node-1` | Nœud Worker 1 | `10.10.0.31` | `52:54:00:80:0b:ca` | Ubuntu 24.04 |
| `K3s-agent-node-2` | Nœud Worker 2 | `10.10.0.32` | `52:54:00:ea:e4:26` | Ubuntu 24.04 |
| `K3s-agent-node-3` | Nœud Worker 3 | `10.10.0.33` | `52:54:00:e2:f2:2b` | Ubuntu 24.04 |

### Installation K3s sur srv-3

```bash
curl -sfL https://get.k3s.io | K3S_TOKEN="<token-de-srv-1>" sh -s - server \
  --datastore-endpoint="postgres://k3s:k3s_password@10.10.0.20:5432/k3s" \
  --tls-san 10.10.0.10 \
  --tls-san 10.10.0.13
```

---

## Vérification

```bash
kubectl get nodes
```

```
NAME               STATUS   ROLES           AGE   VERSION
k3s-agent-node-1   Ready    <none>          19d   v1.35.5+k3s1
k3s-agent-node-2   Ready    <none>          19d   v1.35.5+k3s1
k3s-agent-node-3   Ready    <none>          19d   v1.35.5+k3s1
k3s-srv-1          Ready    control-plane   19d   v1.35.5+k3s1
k3s-srv-2          Ready    control-plane   19d   v1.35.5+k3s1
k3s-srv-3          Ready    control-plane   13m   v1.35.5+k3s1
```

---
![Added serveur 3](img/addde-srv3.png)
## Prochaine étape

Migration du datastore PostgreSQL → etcd embarqué.
Avec 3 nœuds serveurs, le cluster a le quorum nécessaire pour etcd.
