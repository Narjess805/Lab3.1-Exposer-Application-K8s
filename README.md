<div align="center">

# Lab 3.1 : Exposer une application sur Kubernetes (K3s)

ClusterIP, NodePort et LoadBalancer : diagnostic, DNS et répartition de charge

![Kubernetes](https://img.shields.io/badge/Kubernetes-K3s-326CE5?logo=kubernetes&logoColor=white)
![K3s](https://img.shields.io/badge/K3s-Lightweight-FFC61C?logo=rancher&logoColor=black)
![Ubuntu](https://img.shields.io/badge/OS-Ubuntu-E95420?logo=ubuntu&logoColor=white)
![VirtualBox](https://img.shields.io/badge/Hyperviseur-VirtualBox-183A61?logo=virtualbox&logoColor=white)
![Status](https://img.shields.io/badge/Statut-Termin%C3%A9-success)

</div>

---

## Table des matières

1. [Description du projet](#1-description-du-projet)
2. [Environnement de travail](#2-environnement-de-travail)
3. [Structure du dépôt](#3-structure-du-dépôt)
4. [Étapes réalisées et résultats](#4-étapes-réalisées-et-résultats)
5. [Questions de réflexion](#5-questions-de-réflexion)
6. [Auteur](#6-auteur)

---

## 1. Description du projet

Ce dépôt contient le compte rendu complet du **Lab 3.1**, dont l'objectif est de déployer une application sur un cluster Kubernetes (K3s) et de l'exposer de trois manières : **ClusterIP**, **NodePort** et **LoadBalancer**.

L'application utilisée est **Podinfo**, une petite application web qui renvoie le nom du Pod ayant traité la requête. Cela permet d'observer visuellement la répartition de charge (*Load Balancing*) entre les différents Pods.

```mermaid
flowchart LR
    subgraph Externe
        C([Client])
    end
    subgraph Cluster K3s
        LB[LoadBalancer<br/>podinfo-lb : 8080]
        NP[NodePort<br/>podinfo-nodeport : 32243]
        CIP[ClusterIP<br/>podinfo-svc : 80]
        P1[Pod 1 : 9898]
        P2[Pod 2 : 9898]
        P3[Pod 3 : 9898]
    end
    C --> LB --> CIP
    C --> NP --> CIP
    CIP --> P1
    CIP --> P2
    CIP --> P3
```

## 2. Environnement de travail

| Composant | Détails |
|-----------|---------|
| Cluster | K3s v1.36.5 |
| Topologie | 1 Master (control-plane) + 2 Workers |
| Hyperviseur | Oracle VirtualBox |
| OS | Ubuntu 22.04.5 LTS (Master) / Ubuntu 24.04.2 LTS (Workers) |

**Réseau (Host-Only)**

| Nœud | Rôle | Adresse IP |
|------|------|------------|
| `master-2` | Master | `192.168.56.20` |
| `k3s-worker1` | Worker | `192.168.56.31` |
| `k3s-worker2` | Worker | `192.168.56.32` |

## 3. Structure du dépôt

```text
.
├── README.md                  # Documentation du lab
├── yaml/                      # Manifestes Kubernetes
│   ├── podinfo.yaml           # Deployment (3 réplicas)
│   ├── podinfo-svc.yaml       # Service ClusterIP
│   ├── podinfo-nodeport.yaml  # Service NodePort
│   └── podinfo-lb.yaml        # Service LoadBalancer
└── captures/                  # Captures d'écran (preuves d'exécution)
    ├── 01_pods_running.png
    ├── 02_service_endpoints.png
    ├── 03_dns_test_json.png
    ├── 04_load_balancing.png
    ├── 05_endpoints_watch.png
    ├── 06_diagnostic_panne.png
    ├── 07_nodeport_test.png
    ├── 08_loadbalancer_svclb.png
    ├── 09_namespace_dns.png
    └── 10_nettoyage.png
```

## 4. Étapes réalisées et résultats

### Étape 1 : Préparer le namespace et déployer l'application

Création du namespace `tp-k8s` et déploiement de 3 réplicas de Podinfo.

```bash
kubectl create namespace tp-k8s
kubectl config set-context --current --namespace=tp-k8s
kubectl apply -f yaml/podinfo.yaml
kubectl get pods -o wide
```

> **Résultat** : 3 Pods `podinfo` sont en statut `Running`, répartis sur les 3 nœuds du cluster.

![Pods en cours d'exécution](captures/01_pods_running.png)

---

### Étape 2 : Créer un Service ClusterIP

Création d'un Service interne qui écoute sur le port `80` et redirige vers le port `9898` des conteneurs.

```bash
kubectl apply -f yaml/podinfo-svc.yaml
kubectl get service podinfo-svc
kubectl get endpoints podinfo-svc
```

> **Résultat** : le Service obtient la `CLUSTER-IP` `10.43.158.104`. Les endpoints listent bien les 3 adresses IP des Pods.

![Service et endpoints](captures/02_service_endpoints.png)

---

### Étape 3 : Joindre le Service par son nom DNS

Test de résolution DNS depuis un Pod temporaire (`busybox`).

```bash
kubectl run test --rm -it --image=busybox:1.36 --restart=Never -- wget -qO- http://podinfo-svc
kubectl run test --rm -it --image=busybox:1.36 --restart=Never -- wget -qO- http://podinfo-svc.tp-k8s.svc.cluster.local
```

> **Résultat** : les deux appels (nom court et FQDN) renvoient le document JSON de Podinfo contenant le champ `"hostname"`.

![Test DNS et réponse JSON](captures/03_dns_test_json.png)

---

### Étape 4 : Observer la répartition de charge

Envoi de 6 requêtes successives pour observer la distribution du trafic entre les Pods.

```bash
kubectl run test --rm -it --image=busybox:1.36 --restart=Never -- sh -c 'for i in 1 2 3 4 5 6; do wget -qO- http://podinfo-svc | grep hostname; done'
```

> **Résultat** : les réponses affichent des noms de Pods différents (ex : `pjmjt`, `d7pnw`), ce qui prouve le Load Balancing.

![Répartition de charge](captures/04_load_balancing.png)

---

### Étape 5 : Suivre les Pods avec les endpoints

Observation en temps réel de la mise à jour des endpoints lors d'un scale-up et de la suppression d'un Pod.

```bash
# Terminal 1
kubectl get endpoints podinfo-svc -w

# Terminal 2
kubectl scale deployment podinfo --replicas=5
kubectl delete pod <nom-du-pod>
kubectl scale deployment podinfo --replicas=3
```

> **Résultat** : les endpoints passent de 3 à 5 adresses, puis se mettent à jour dynamiquement lors de la suppression du Pod.

![Suivi des endpoints en direct](captures/05_endpoints_watch.png)

---

### Étape 6 : Diagnostiquer un sélecteur incorrect

Simulation d'une panne en modifiant le sélecteur du Service.

```bash
kubectl set selector service podinfo-svc app=autre
kubectl get endpoints podinfo-svc
kubectl describe service podinfo-svc | grep Selector

# Réparation
kubectl set selector service podinfo-svc app=podinfo
```

> **Résultat** : les endpoints deviennent `<none>` et l'appel échoue (`Connection refused`). Après réparation, les 3 endpoints sont restaurés.

![Diagnostic de panne](captures/06_diagnostic_panne.png)

---

### Étape 7 : Exposer hors du cluster avec NodePort

Création d'un Service de type NodePort pour exposer l'application sur un port de chaque nœud.

```bash
kubectl apply -f yaml/podinfo-nodeport.yaml
kubectl get service podinfo-nodeport
curl http://192.168.56.20:32243
curl http://192.168.56.31:32243
```

> **Résultat** : l'application répond sur le port `32243` (NodePort), sur n'importe quel nœud, même ceux qui n'hébergent aucun Pod.

![Test du NodePort](captures/07_nodeport_test.png)

---

### Étape 8 : Exposer avec LoadBalancer (ServiceLB)

Création d'un Service de type LoadBalancer via le contrôleur ServiceLB intégré à K3s.

```bash
kubectl apply -f yaml/podinfo-lb.yaml
kubectl get service podinfo-lb
kubectl get pods -n kube-system | grep svclb
```

> **Résultat** : le Service obtient une `EXTERNAL-IP` (les IP des nœuds). Des Pods `svclb-podinfo-lb-...` sont créés dans le namespace `kube-system` et agissent comme proxys sur le port `8080`.

![LoadBalancer et pods svclb](captures/08_loadbalancer_svclb.png)

---

### Étape 9 : Joindre un Service depuis un autre namespace

Test de la résolution DNS depuis le namespace `tp-autre`.

```bash
kubectl create namespace tp-autre
kubectl run test -n tp-autre --rm -it --image=busybox:1.36 --restart=Never -- wget -T 3 -qO- http://podinfo-svc
kubectl run test -n tp-autre --rm -it --image=busybox:1.36 --restart=Never -- wget -qO- http://podinfo-svc.tp-k8s
```

> **Résultat** : le nom court échoue (`bad address`), mais le nom qualifié avec le namespace (`podinfo-svc.tp-k8s`) réussit.

![DNS entre namespaces](captures/09_namespace_dns.png)

---

### Étape 10 : Nettoyage

```bash
kubectl delete namespace tp-autre
kubectl delete namespace tp-k8s
kubectl config set-context --current --namespace=default
```

> **Résultat** : l'environnement est propre, le cluster est prêt pour le prochain TP.

![Nettoyage de l'environnement](captures/10_nettoyage.png)

## 5. Questions de réflexion

<details>
<summary><strong>1. Quelle est la différence entre <code>port</code> et <code>targetPort</code> dans un Service ?</strong></summary>

- `port` : le port sur lequel le Service écoute (ex : `80`).
- `targetPort` : le port du conteneur vers lequel le trafic est transmis (ex : `9898`).
</details>

<details>
<summary><strong>2. Pourquoi l'application répond-elle sur l'adresse d'un nœud qui n'héberge aucun Pod ?</strong></summary>

Un NodePort est ouvert sur **tous** les nœuds du cluster. `kube-proxy` intercepte le trafic sur ce port et le redirige vers un Pod sain du Service, quel que soit le nœud qui l'héberge.
</details>

<details>
<summary><strong>3. Qu'apprend-on d'une liste d'endpoints vide ?</strong></summary>

Aucun Pod prêt ne correspond au sélecteur du Service. Causes possibles : sélecteur erroné, labels différents, Pods absents ou en erreur. C'est le premier contrôle à faire quand un Service ne répond pas.
</details>

<details>
<summary><strong>4. Pourquoi ne pas exposer toutes les applications web avec un NodePort ?</strong></summary>

Chaque application consomme un port élevé distinct, ce qui est difficile à gérer et à sécuriser. De plus, un NodePort ne permet pas le routage par nom d'hôte. C'est le rôle de l'Ingress (Lab 3.2).
</details>

<details>
<summary><strong>5. Qu'est-ce qui distingue ClusterIP, NodePort et LoadBalancer ?</strong></summary>

| Type | Rôle |
|------|------|
| **ClusterIP** | Adresse interne uniquement, accessible depuis le cluster |
| **NodePort** | Ouvre un port statique sur chaque nœud pour un accès externe basique |
| **LoadBalancer** | Ajoute une adresse externe (fournisseur cloud, ou ServiceLB dans K3s) pour un accès externe transparent |
</details>

## 6. Auteur

| | |
|---|---|
| **Nom** | Dabbech |
| **Prénom** | Narjess |
| **Classe** | M2 DevOps |
| **GitHub** | [@Narjess805](https://github.com/Narjess805) |
| **Date** | Octobre 2026 |
