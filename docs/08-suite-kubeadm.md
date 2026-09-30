# 08 — Suite : du lab K3s au cluster kubeadm

Ce lab K3s a servi à apprendre : haute disponibilité du control plane, datastore externe, load balancing, puis supervision et segmentation réseau autour du cluster (voir [K3s-lab-monitoring](https://github.com/Souheib-h/K3s-lab-monitoring)). Il est **figé** : les corrections ne sont plus appliquées ici mais sur un cluster **kubeadm** (Calico, préparation CKA), sur le réseau `cluster-ha-net` (`10.40.0.0/24`).

Cette page est la checklist de départ de ce nouveau cluster. Chaque point vient d'un problème **constaté et vérifié** sur ce lab (audit de septembre 2026).

---

## Sécurité Kubernetes

| # | À faire sur kubeadm | Constaté sur K3s |
|---|---|---|
| K1 | RBAC de scraping Prometheus limité à `get` sur `nodes/metrics` | Le ClusterRole incluait `nodes/proxy` (`kubectl auth can-i get nodes/proxy` → `yes`) : un token volé ouvre `/exec`, `/run` et `/pods` du kubelet |
| K2 | Scraping des kubelets avec `ca_file` (CA du cluster) | `insecure_skip_verify: true` : le token est envoyé à quiconque répond sur `:10250` |
| K3 | Aucun secret en ligne de commande ni dans le repo ; chiffrement des Secrets au repos (`EncryptionConfiguration`) | Mot de passe du datastore dans `k3s.service` (lisible par tous) et dans les docs 02, 04 et 07 |
| K4 | etcd empilé sur 3 control planes, **sauvegardes `etcdctl snapshot` planifiées et testées** (restauration comprise) | PostgreSQL unique (`K3s-db`) : point unique de défaillance ; pas de migration possible sur place vers etcd |
| K5 | **Tous** les control planes derrière le load balancer ; LB doublé (keepalived + VIP) ; page de stats avec un vrai mot de passe | `srv3` absent du backend HAProxy ; un seul `Load-srvs` ; stats en `admin:admin` |
| K6 | Même image d'OS et même version sur tous les nœuds | Ubuntu 24.04 sur 5 nœuds, 26.04 sur srv-3 |
| K7 | Manifests durcis par défaut : images figées, non-root, requests/limits, probes, capabilities retirées | Voir [`manifests/test/nginx.yaml`](../manifests/test/nginx.yaml), durci après coup |

## Réseau (`cluster-ha-net`, 10.40.0.0/24)

| # | À faire | Constaté sur k3s-net |
|---|---|---|
| N1 | Règles OPNsense sur l'interface `CLUSTERHANET` limitées aux flux nécessaires (agents Wazuh/Zabbix/Alloy, ping vers OPNsense, Internet hors réseaux privés), **jamais `net → any`** | Une règle `k3snet net → any` laissait le cluster joindre l'API Wazuh, l'indexer, Loki et le bastion ; remplacée par 5 règles (K3s-lab-monitoring, ADR-016 amendement) |
| N2 | Option DHCP 121 de `cluster-ha-net` déclarée sur **une seule ligne**, route par défaut comprise si les nœuds sortent via OPNsense | dnsmasq ne gardait que la dernière des 3 lignes, et l'option 121 sans route par défaut privait les nœuds de route par défaut (RFC 3442), ce qui empêchait K3s de démarrer (ADR-017) |
| N3 | Routes statiques dans `fix-routes.yml` (fichier canonique unique), route par défaut via OPNsense comprise | Un playbook hors repo ajoutait la route par défaut dans un fichier que `fix-routes.yml` réécrivait sans elle |

## Supervision

| # | À faire | Référence |
|---|---|---|
| M1 | Prometheus : cibles kubelet, cAdvisor et kube-state-metrics vers les nœuds kubeadm ; kube-state-metrics sur **une seule** cible | [Scraping Prometheus → K3s](https://github.com/Souheib-h/K3s-lab-monitoring/blob/main/docs/phase-5-ansible/prometheus-k3s-scraping.md) |
| M2 | Inventaire Ansible : groupe pour les nœuds kubeadm, agents Zabbix / Wazuh / Alloy déployés par les playbooks existants | [`configs/ansible`](https://github.com/Souheib-h/K3s-lab-monitoring/tree/main/configs/ansible) |
| M3 | `health-check.yml` : contrôle d'isolation du bastion étendu aux nœuds de `cluster-ha-net` | ADR-016 amendement |
