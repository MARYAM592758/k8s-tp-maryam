# Compte rendu TP 4 : Kubernetes

**Virtualisation et cloud computing** : orchestration de conteneurs avec Kubernetes (cluster multi-nœuds avec kind, Deployment, Service, rolling update et rollback)

- Université Abdelmalek Essaadi, École Nationale des Sciences Appliquées Al Hoceima (ENSAH)
- Filière : Élève-ingénieur en 5e année du cycle d'ingénieur en génie informatique
- Réalisé par : Dafali Maryam
- Encadrante : Mme Hayat ROUTAIB
- Année universitaire : 2026 / 2027

## Sommaire

1. [Installation et vérification des outils](#1-installation-et-vérification-des-outils)
2. [Partie A : création du cluster](#2-partie-a--création-du-cluster)
3. [Partie B : le Pod](#3-partie-b--le-pod)
4. [Partie C : Deployment et Service](#4-partie-c--deployment-et-service)
5. [Partie D : rolling update et rollback](#5-partie-d--rolling-update-et-rollback)
6. [Réponses aux questions](#6-réponses-aux-questions)
7. [Défi : joindre le Service par son nom DNS](#7-défi--joindre-le-service-par-son-nom-dns)
8. [Fichiers rendus](#8-fichiers-rendus)

## 1. Installation et vérification des outils

Le TP demande quatre outils : Docker (moteur de conteneurs), `kubectl` (client de l'API Kubernetes), kind (Kubernetes in Docker : chaque nœud du cluster est un conteneur Docker) et Helm (gestionnaire de paquets Kubernetes). Tout est installé sous Windows avec WSL2 (Ubuntu).

### 1.1 Docker

```bash
docker run hello-world
```

Le message « Hello from Docker! » confirme que le client contacte le démon, que l'image `hello-world` (amd64) est téléchargée depuis Docker Hub, qu'un conteneur est créé, puis que sa sortie est renvoyée au terminal : l'installation fonctionne.

### 1.2 kubectl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```

La commande télécharge la dernière version stable du binaire `kubectl` (59 Mo, en 10 s).

```bash
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
rm kubectl
```

Le binaire est installé dans `/usr/local/bin` (propriétaire root, droits 0755) : il est donc accessible depuis n'importe quel dossier. Le fichier téléchargé est ensuite supprimé.

### 1.3 kind

```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.24.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

On télécharge kind v0.24.0, on le rend exécutable avec `chmod +x`, puis on le déplace dans `/usr/local/bin`.

### 1.4 Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

Le script officiel télécharge Helm v3.22.1, vérifie la somme de contrôle (« Verifying checksum... Done. ») et installe le binaire dans `/usr/local/bin/helm`.

### 1.5 Dossier de travail

```bash
cd ~
mkdir k8s-tp-maryam
cd k8s-tp-maryam
```

Le dossier `k8s-tp-maryam` regroupe tous les fichiers du TP ; il correspond au dépôt GitHub public du même nom.

## 2. Partie A : création du cluster

Le cluster est décrit dans le fichier `kind-config.yaml` : un nœud control-plane et deux workers. Le port 30080 du conteneur du control-plane est publié sur le port 30080 de la machine hôte (`extraPortMappings`), ce qui permet d'atteindre un Service NodePort avec `localhost:30080`.

```bash
ls
cat kind-config.yaml
```

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: tp-k8s
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 30080
        hostPort: 30080
  - role: worker
  - role: worker
```

### 2.1 Création du cluster

```bash
kind create cluster --config kind-config.yaml
```

kind télécharge l'image de nœud `kindest/node:v1.31.0`, prépare les nœuds, écrit la configuration, démarre le control-plane, installe le CNI et la StorageClass, puis joint les deux workers. Le contexte kubectl `kind-tp-k8s` est configuré automatiquement.

### 2.2 Les nœuds

```bash
kubectl get nodes -o wide
```

Le cluster compte trois nœuds en version v1.31.0, avec le runtime containerd 1.7.18. Lors de la première exécution (22 s après la création), `tp-k8s-worker2` est encore `NotReady` : le kubelet et le réseau du nœud finissent de s'initialiser. Quelques secondes plus tard, les trois nœuds sont `Ready`.

| Nœud | Rôle | IP interne |
| --- | --- | --- |
| tp-k8s-control-plane | control-plane | 172.18.0.2 |
| tp-k8s-worker | worker | 172.18.0.4 |
| tp-k8s-worker2 | worker | 172.18.0.3 |

### 2.3 Les Pods système

```bash
kubectl get pods -n kube-system
```

Le namespace `kube-system` contient les composants du cluster, tous en `Running` sans redémarrage : 2 CoreDNS, etcd, kube-apiserver, kube-controller-manager, kube-scheduler, 3 kindnet et 3 kube-proxy (un de chaque par nœud). Leur rôle est détaillé à la question 1.

### 2.4 Les nœuds sont des conteneurs Docker

```bash
docker ps
```

Chaque nœud Kubernetes est un conteneur Docker basé sur l'image `kindest/node:v1.31.0`. Le conteneur du control-plane publie `0.0.0.0:30080->30080/tcp` (le port défini dans la configuration) et `127.0.0.1:35887->6443/tcp` (l'API server, utilisé par kubectl).

## 3. Partie B : le Pod

Un Pod est la plus petite unité déployable de Kubernetes : il regroupe un ou plusieurs conteneurs qui partagent le réseau (une IP unique) et le stockage. On crée ici un Pod isolé, sans contrôleur, pour observer la différence avec un Deployment.

### 3.1 Création et inspection

```bash
kubectl run web --image=nginx:1.27 --port=80
```

La commande crée le Pod `web` à partir de l'image `nginx:1.27`, avec le port 80 déclaré.

```bash
kubectl get pods -o wide
```

Le Pod est `Running` (1/1, 0 redémarrage), avec l'IP 10.244.2.2, planifié sur le nœud `tp-k8s-worker2`.

```bash
kubectl describe pod web
```

La description donne le nœud (`tp-k8s-worker2/172.18.0.3`), l'étiquette `run=web`, l'image `nginx:1.27`, le port 80/TCP, l'état `Running`, les conditions (`Ready`, `PodScheduled`…) et la classe de QoS `BestEffort` (aucune ressource demandée).

La section Events retrace le cycle de vie du Pod : planifié par le `default-scheduler` sur `worker2`, image téléchargée par le kubelet (72 Mo en 38,7 s), conteneur créé puis démarré.

```bash
kubectl logs web
```

Les journaux montrent les scripts d'entrée de l'image nginx (configuration IPv6 activée, « Configuration complete; ready for start up »), puis nginx/1.27.5 qui démarre ses processus workers.

### 3.2 Accès au Pod

```bash
kubectl exec -it web -- sh
```

La commande ouvre un shell dans le conteneur. `hostname` renvoie `web`, le nom du Pod.

```bash
kubectl port-forward pod/web 8080:80
```

Le port 8080 de la machine locale est redirigé vers le port 80 du Pod (« Handling connection for 8080 »). En ouvrant `http://localhost:8080` dans le navigateur, la page « Welcome to nginx! » s'affiche : le serveur fonctionne.

### 3.3 Suppression : le Pod n'est pas recréé

```bash
kubectl delete pod web
kubectl get pods
```

Après la suppression, `kubectl get pods` répond « No resources found in default namespace » : aucun contrôleur ne surveille ce Pod, il disparaît définitivement (voir question 2).

### 3.4 Passage au déclaratif

```bash
kubectl run web --image=nginx:1.27 --dry-run=client -o yaml > pod.yaml
cat pod.yaml
```

L'option `--dry-run=client` génère le manifeste sans rien créer ; `-o yaml` l'affiche en YAML et la redirection l'enregistre dans `pod.yaml`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: web
  name: web
spec:
  containers:
  - image: nginx:1.27
    name: web
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

Le manifeste contient la version d'API (`v1`), le type (`Pod`), les métadonnées (nom `web` et étiquette `run=web`) et la spécification : un conteneur nginx:1.27, la politique DNS `ClusterFirst` et `restartPolicy: Always` (le kubelet relance le conteneur s'il s'arrête).

```bash
kubectl apply -f pod.yaml
kubectl get pods
```

Le Pod `web` est recréé à partir du fichier (`pod/web created`) et passe à `Running` (1/1). Le fichier décrit l'état voulu : c'est l'approche déclarative.

```bash
kubectl delete -f pod.yaml
```

Le Pod est supprimé à partir du même fichier avant de passer à la partie C.

## 4. Partie C : Deployment et Service

Un Deployment maintient en permanence le nombre voulu de Pods identiques (par l'intermédiaire d'un ReplicaSet) ; un Service donne à ces Pods une adresse stable et répartit le trafic entre eux. Le fichier `whoami.yaml` définit les deux objets : 3 réplicas de l'image `traefik/whoami:v1.10`, exposés par un Service NodePort sur le port 30080.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: whoami
spec:
  replicas: 3
  selector:
    matchLabels:
      app: whoami
  template:
    metadata:
      labels:
        app: whoami
    spec:
      containers:
      - name: whoami
        image: traefik/whoami:v1.10
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: whoami
spec:
  type: NodePort
  selector:
    app: whoami
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
```

### 4.1 Déploiement

```bash
kubectl apply -f whoami.yaml
```

Le Deployment et le Service sont créés (`deployment.apps/whoami created`, `service/whoami created`).

```bash
kubectl get deploy,rs,pods,svc -o wide
```

La commande affiche toute la chaîne d'objets : le Deployment `whoami` (3/3 prêts, image `traefik/whoami:v1.10`), le ReplicaSet `whoami-85f6bb67d6` (3 souhaités, 3 courants, 3 prêts), les trois Pods en `Running` et le Service `whoami` de type NodePort (IP 10.96.144.184, ports 80:30080/TCP). Le scheduler a réparti les Pods sur les deux workers : deux sur `tp-k8s-worker` et un sur `tp-k8s-worker2`. Le suffixe `85f6bb67d6` des noms de Pods est le hash du modèle de Pod, repris du ReplicaSet.

### 4.2 Accès et répartition de charge

```bash
curl http://localhost:30080
```

Le port 30080 de l'hôte atteint le Service grâce au `extraPortMappings` de kind. Le premier appel renvoie `Hostname: whoami-85f6bb67d6-tx65n` (IP 10.244.2.3). Le second appel renvoie un autre Pod, `whoami-85f6bb67d6-p4qtw` (IP 10.244.1.3) : le Service répartit les requêtes entre les Pods. Le `RemoteAddr` 172.18.0.2 est l'adresse du nœud control-plane, qui relaie la requête.

### 4.3 Auto-réparation

```bash
kubectl delete pod whoami-85f6bb67d6-tx65n
kubectl get pods
```

Après la suppression de `tx65n`, un nouveau Pod `whoami-85f6bb67d6-jsh7p` apparaît (âgé de 8 s) alors que les deux autres ont 3 min 4 s : le ReplicaSet a recréé immédiatement le Pod manquant pour revenir à 3 réplicas. C'est la différence avec le Pod isolé `web` de la partie B.

### 4.4 Mise à l'échelle

```bash
kubectl scale deployment whoami --replicas=6
kubectl get pods
```

Le Deployment passe à 6 réplicas : trois nouveaux Pods (âgés de 3 s) rejoignent les trois existants, tous `Running` et prêts.

### 4.5 Les EndpointSlices

```bash
kubectl get endpointslices
kubectl describe endpointslice whoami-9t44j
```

L'EndpointSlice `whoami-9t44j` est créée automatiquement pour le Service `whoami` ; elle liste les 6 adresses des Pods sur le port 80. La description montre l'étiquette `kubernetes.io/service-name=whoami` et, pour chaque endpoint, l'adresse, la condition `Ready: true`, le Pod ciblé (`TargetRef`) et son nœud. Le détail est repris à la question 4.

## 5. Partie D : rolling update et rollback

Une mise à jour continue (rolling update) remplace progressivement les anciens Pods par de nouveaux, sans interrompre le service. Le Deployment compte ici 6 réplicas, avec la stratégie `RollingUpdate` par défaut (25 % d'indisponibilité maximale et 25 % de surplus maximal).

### 5.1 Mise à jour vers la version v1.11

```bash
kubectl set image deployment/whoami whoami=traefik/whoami:v1.11
```

La commande change l'image du conteneur `whoami` et déclenche la mise à jour progressive (`deployment.apps/whoami image updated`).

```bash
kubectl rollout status deployment/whoami
```

Le suivi affiche la progression : « 3 out of 6 new replicas have been updated », puis 4, puis 5, puis « old replicas are pending termination » (3, 2, puis 1 ancien Pod restant) et enfin « deployment "whoami" successfully rolled out ». Les anciens Pods ne sont supprimés qu'à mesure que les nouveaux deviennent prêts.

```bash
kubectl rollout history deployment/whoami
```

L'historique contient deux révisions : la 1 (image v1.10) et la 2 (image v1.11). La colonne `CHANGE-CAUSE` est vide (`<none>`) car aucune annotation `kubernetes.io/change-cause` n'a été renseignée.

### 5.2 Mise à jour échouée (image inexistante)

```bash
kubectl set image deployment/whoami whoami=traefik/whoami:v9.99
kubectl get pods
```

L'image `v9.99` n'existe pas ; Kubernetes accepte tout de même la modification et lance un nouveau déploiement. Deux ReplicaSets coexistent : l'ancien (`67dc58c95f`, v1.11) a toujours 5 Pods `Running`, tandis que le nouveau (`7cd8b66f95`, v9.99) a 3 Pods qui ne démarrent pas : un en `ContainerCreating` et deux en `ErrImagePull` (0/1), car l'image est introuvable dans le registre. Après quelques nouvelles tentatives, le statut devient `ImagePullBackOff` (attente de plus en plus longue entre deux essais). Les 5 Pods de la version précédente restent actifs.

### 5.3 Retour arrière

```bash
kubectl rollout undo deployment/whoami
```

Le Deployment revient à la révision précédente (image v1.11). Kubernetes affiche un avertissement : la ressource a été créée avec `kubectl apply`, et le rollback ne met pas à jour l'annotation `last-applied-configuration`, ce qui pourrait surprendre lors d'un futur `apply`. Le message `deployment.apps/whoami rolled back` confirme l'opération.

```bash
kubectl rollout status deployment/whoami
kubectl get rs
```

La commande confirme « successfully rolled out » : les 6 Pods sont de nouveau disponibles. Trois ReplicaSets existent encore, mais un seul est actif : `67dc58c95f` (v1.11) avec 6/6/6, alors que `7cd8b66f95` (v9.99) et `85f6bb67d6` (v1.10) sont à 0.

### 5.4 Historique final

```bash
kubectl rollout history deployment/whoami > rollout-history.txt
cat rollout-history.txt
```

```text
deployment.apps/whoami
REVISION  CHANGE-CAUSE
1         <none>
3         <none>
4         <none>
```

La révision 2 a disparu : le rollback a réactivé la configuration de la révision 2 (v1.11), qui est renumérotée en révision 4. Les révisions restantes sont donc la 1 (v1.10), la 3 (v9.99, cassée) et la 4 (v1.11, active). La sortie est enregistrée dans `rollout-history.txt`, un des livrables demandés.

## 6. Réponses aux questions

### Question 1 : rôle de chaque Pod de kube-system

| Composant | Où | Rôle |
| --- | --- | --- |
| etcd | control-plane | Base de données clé-valeur qui stocke tout l'état du cluster (objets, configuration) : c'est la source de vérité. |
| kube-apiserver | control-plane | Point d'entrée unique : il expose l'API REST utilisée par kubectl et par tous les composants, valide les requêtes et lit/écrit dans etcd. |
| kube-scheduler | control-plane | Choisit le nœud où chaque nouveau Pod sera exécuté, selon les ressources disponibles et les contraintes (taints, affinités). |
| kube-controller-manager | control-plane | Exécute les contrôleurs (Deployment, ReplicaSet, Node, EndpointSlice…) : chacun compare l'état voulu à l'état réel et corrige l'écart. |
| kube-proxy | un par nœud (3) | Programme les règles réseau (iptables/IPVS) qui relient l'IP virtuelle d'un Service aux IP des Pods qui le composent. |
| CoreDNS | 2 réplicas | Serveur DNS du cluster : résout les noms de Services et de Pods (ex. `whoami.default.svc.cluster.local`). |
| kindnet | un par nœud (3) | Plugin réseau (CNI) de kind : donne une IP à chaque Pod et assure le routage entre Pods de nœuds différents. |

### Question 2 : pourquoi le Pod `web` n'est-il pas recréé ?

Le Pod `web` a été créé seul, sans contrôleur. Quand on le supprime, personne ne compare l'état réel à un état voulu : il disparaît définitivement (« No resources found »). `restartPolicy: Always` ne s'applique qu'au conteneur à l'intérieur d'un Pod existant (le kubelet le relance s'il plante), pas à la suppression du Pod lui-même.

Les Pods d'un Deployment sont gérés par un ReplicaSet dont le rôle est de maintenir le nombre de réplicas demandé. Après la suppression de `whoami-85f6bb67d6-tx65n`, il constate 2 Pods au lieu de 3 et en crée un nouveau (`jsh7p`).

### Question 3 : Deployment, ReplicaSet et Pod

La relation est une hiérarchie de propriété : **Deployment → ReplicaSet → Pods**.

- Le Deployment décrit l'application (modèle de Pod, nombre de réplicas, stratégie de mise à jour) et gère les versions.
- Il crée un ReplicaSet par version du modèle de Pod ; chaque ReplicaSet maintient le nombre voulu de Pods de sa version.
- Les Pods exécutent les conteneurs ; leur nom reprend celui du ReplicaSet (`whoami-85f6bb67d6-p4qtw`).

À la fin du TP, **trois ReplicaSets** existent : `85f6bb67d6` (v1.10), `7cd8b66f95` (v9.99) et `67dc58c95f` (v1.11). Chaque changement d'image a créé un ReplicaSet, et les anciens sont conservés à 0 réplica pour permettre le retour arrière (`revisionHistoryLimit`, 10 par défaut). Seul `67dc58c95f` est actif (6 Pods) : le `rollout undo` a réutilisé ce ReplicaSet existant au lieu d'en créer un quatrième.

### Question 4 : comment le Service trouve-t-il ses Pods ?

Le Service `whoami` porte le sélecteur `app=whoami`. Le contrôleur EndpointSlice (dans kube-controller-manager) cherche en continu les Pods portant cette étiquette et dans l'état `Ready`, puis maintient l'objet EndpointSlice `whoami-9t44j` avec leurs adresses. kube-proxy lit cet objet pour répartir le trafic entre ces adresses.

| Adresse | Pod | Nœud |
| --- | --- | --- |
| 10.244.1.3 | whoami-85f6bb67d6-p4qtw | tp-k8s-worker |
| 10.244.1.4 | whoami-85f6bb67d6-w6snv | tp-k8s-worker |
| 10.244.1.5 | whoami-85f6bb67d6-nbmsd | tp-k8s-worker |
| 10.244.2.4 | whoami-85f6bb67d6-jsh7p | tp-k8s-worker2 |
| 10.244.2.5 | whoami-85f6bb67d6-nm2ll | tp-k8s-worker2 |
| 10.244.2.6 | whoami-85f6bb67d6-2ww76 | tp-k8s-worker2 |

La liste suit les Pods automatiquement : lorsqu'un Pod est supprimé ou remplacé, son adresse sort de la liste et celle du nouveau Pod y entre dès qu'il est prêt. Le Service n'a donc jamais besoin de connaître les noms des Pods.

### Question 5 : l'application est-elle restée disponible avec l'image cassée ?

Oui. Avec 6 réplicas, la stratégie par défaut donne :

- `maxUnavailable` = 25 % de 6 = 1,5, arrondi à l'inférieur : **1 Pod** au plus peut être indisponible, donc au moins 5 Pods doivent rester disponibles ;
- `maxSurge` = 25 % de 6 = 1,5, arrondi au supérieur : **2 Pods** de plus que le nombre voulu peuvent exister, soit 8 au maximum.

La capture du `get pods` correspond exactement à ces limites : 5 anciens Pods `Running` (v1.11) et 3 nouveaux Pods (v9.99) bloqués en `ErrImagePull` / `ContainerCreating`. Comme les nouveaux Pods ne deviennent jamais `Ready`, le contrôleur ne peut supprimer qu'un seul ancien Pod et la mise à jour reste bloquée. Le Service n'envoie du trafic qu'aux Pods `Ready` : les 5 anciens Pods restaient donc disponibles pour servir les requêtes. Le `rollout undo` a ensuite rétabli les 6 réplicas en v1.11.

## 7. Défi : joindre le Service par son nom DNS

On lance un Pod temporaire dans le cluster pour tester la résolution de noms.

```bash
kubectl run test --rm -it --image=busybox:1.36 -- sh
```

Le Pod `test` démarre avec un shell ; `--rm` le supprime à la sortie.

```sh
nslookup whoami.default.svc.cluster.local
```

Le serveur DNS interrogé est `10.96.0.10:53`, le Service CoreDNS du cluster. Il renvoie l'adresse `10.96.144.184`, qui est la ClusterIP du Service `whoami` (la même que dans la partie C).

```sh
wget -qO- http://whoami.default.svc.cluster.local
```

La requête atteint un Pod whoami (`whoami-67dc58c95f-gzvc4`, IP 10.244.1.6). Le `RemoteAddr` 10.244.2.11 est l'IP du Pod `test` lui-même : le trafic reste à l'intérieur du réseau des Pods.

```sh
wget -qO- http://whoami
exit
```

Le nom court fonctionne aussi et tombe sur un autre Pod (`whoami-67dc58c95f-6tsh4`) : le Service répartit la charge. Il fonctionne grâce aux domaines de recherche du fichier `/etc/resolv.conf` du Pod, qui complètent le nom (`default.svc.cluster.local`) ; cela ne marche que depuis le même namespace. À la sortie, le Pod `test` est supprimé (`pod "test" deleted`).

À l'inverse, depuis le terminal WSL (hors du cluster), `nslookup` n'est pas installé et `wget` sur le même nom ne renvoie aucune réponse : ces noms ne sont résolus que par le DNS interne du cluster.

### Explication du nom `whoami.default.svc.cluster.local`

| Partie | Signification |
| --- | --- |
| `whoami` | Nom du Service (`metadata.name` dans `whoami.yaml`). |
| `default` | Namespace dans lequel le Service existe. |
| `svc` | Type d'objet : c'est un Service (les Pods ont leur propre sous-domaine `pod`). |
| `cluster.local` | Domaine DNS du cluster, valeur par défaut de Kubernetes. |

Le format général est `<service>.<namespace>.svc.<domaine-du-cluster>`. Un Service d'un autre namespace s'atteint donc avec `<service>.<autre-namespace>`.

## 8. Fichiers rendus

Dépôt GitHub : [MARYAM592758/k8s-tp-maryam](https://github.com/MARYAM592758/k8s-tp-maryam)

| Fichier | Contenu |
| --- | --- |
| `kind-config.yaml` | Cluster kind : 1 control plane + 2 workers, port 30080 ouvert sur l'hôte. |
| `whoami.yaml` | Deployment (3 réplicas) et Service NodePort 30080. |
| `pod.yaml` | Manifeste du Pod `web` généré avec `--dry-run=client`. |
| `rollout-history.txt` | Sortie de `kubectl rollout history deployment/whoami`. |
| `README.md` | Ce compte rendu, avec les réponses aux questions et le défi. |
