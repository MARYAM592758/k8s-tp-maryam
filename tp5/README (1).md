# TP 5 Kubernetes — Configuration et données persistantes

**ENSA Al Hoceima · Virtualisation et cloud computing · 2026 / 2027**
Réalisé par : **Dafali Maryam** — Encadrante : Mme Hayat ROUTAIB

Objectifs du TP :

- isoler des environnements avec les **Namespaces** ;
- sortir la configuration de l'image avec les **ConfigMap** et les **Secret** ;
- faire survivre les données d'une base PostgreSQL au redémarrage de ses Pods (**PVC**, **StatefulSet**).

Le compte rendu complet (51 captures commentées) est dans `Compte_rendu_TP5_Kubernetes_Dafali_Maryam.pdf`.

---

## Environnement

| Élément | Valeur |
| --- | --- |
| Système | Windows + WSL2 (Ubuntu), Docker Desktop |
| Cluster | kind v0.24.0, cluster `tp-k8s` : 1 control plane + 2 workers |
| Kubernetes | v1.31.0 (kubectl client v1.37.2) |
| Images | `nginx:1.27`, `postgres:17`, `busybox:1.36` |

## Contenu du dossier

| Fichier | Rôle |
| --- | --- |
| `web-config.yaml` | ConfigMap de **dev** (`APP_ENV`, `index.html`) |
| `web-config-prod.yaml` | ConfigMap de **prod** |
| `web-deployment.yaml` | Deployment nginx (variable + volume ConfigMap) et Service `web` |
| `postgres.yaml` | Service headless + StatefulSet PostgreSQL avec `volumeClaimTemplates` |
| `dev-quota.yaml` | ResourceQuota du défi (4 Pods maximum dans `dev`) |

> Le Secret `db-cred` n'est **pas** versionné : il est créé en ligne de commande (voir partie B).
> Le `postgres.yaml` de l'énoncé contient une erreur d'indentation (`value:` sous `PGDATA` et `mountPath:`) ; la version de ce dépôt est corrigée.

---

## Partie A — Namespaces et ConfigMap

```bash
kubectl create namespace dev
kubectl create namespace prod
kubectl config set-context --current --namespace=dev

# dev
kubectl apply -f web-config.yaml
kubectl apply -f web-deployment.yaml
kubectl exec deploy/web -- printenv APP_ENV                          # dev
kubectl exec deploy/web -- cat /usr/share/nginx/html/index.html      # page "environnement dev"

# prod : même manifeste, autre Namespace
kubectl apply -f web-config-prod.yaml
kubectl apply -f web-deployment.yaml -n prod
kubectl exec -n prod deploy/web -- printenv APP_ENV                  # prod
```

**Modification de la ConfigMap de dev.** L'édition avec `kubectl edit cm web-config` a été refusée (`configmaps "web-config" is invalid`, erreur d'indentation). On a donc utilisé l'équivalent non interactif :

```bash
kubectl patch cm web-config --type merge \
  -p '{"data":{"APP_ENV":"dev-v2","index.html":"<h1>ENSAH - GL5 - environnement dev (modifie)</h1>\n"}}'
```

| Élément | Après modification | Pourquoi |
| --- | --- | --- |
| Fichier monté `index.html` | Mis à jour seul (~1 min) | Le kubelet resynchronise les volumes configMap |
| Variable `APP_ENV` | Inchangée (`dev`) | Lue une seule fois, au démarrage du conteneur |

```bash
kubectl rollout restart deployment web
kubectl get rs                                   # ancien RS à 0, nouveau RS à 2
kubectl exec deploy/web -- printenv APP_ENV      # dev-v2
```

`rollout restart` ajoute l'annotation `kubectl.kubernetes.io/restartedAt` au modèle de Pod. Cela déclenche un rolling update, et les nouveaux Pods relisent la ConfigMap.

## Partie B — Secret

```bash
kubectl create secret generic db-cred \
  --from-literal=POSTGRES_USER=etudiant \
  --from-literal=POSTGRES_PASSWORD='S3cret!2026' \
  --from-literal=POSTGRES_DB=ensah
kubectl get secret db-cred -o yaml
echo 'UzNjcmV0ITIwMjY=' | base64 -d     # S3cret!2026
```

Le **base64 n'est pas un chiffrement** : quiconque peut lire le Secret retrouve le mot de passe en une commande.

## Partie C — PostgreSQL en StatefulSet

```bash
kubectl get storageclass            # standard (default), rancher.io/local-path
kubectl apply -f postgres.yaml
kubectl get pvc,pv                  # data-postgres-0 : Bound, 1Gi, RWO

kubectl exec -it postgres-0 -- psql -U etudiant -d ensah \
  -c "CREATE TABLE etudiants(id serial, nom text); INSERT INTO etudiants(nom) VALUES ('Amine'),('Salma');"

kubectl delete pod postgres-0       # le StatefulSet recrée postgres-0 (même nom, même PVC)
kubectl exec -it postgres-0 -- psql -U etudiant -d ensah -c "SELECT * FROM etudiants;"
```

**Preuve de persistance :** après la recréation du Pod, le `SELECT` renvoie toujours `1 | Amine` et `2 | Salma`.

---

## Réponses aux questions

**1. Un Pod de dev peut-il joindre un Service de prod ?**
Oui. Les Namespaces n'isolent **pas** le réseau. Le nom DNS est `web.prod.svc.cluster.local` (forme courte : `web.prod`).

```bash
kubectl run test -n dev --rm -it --image=busybox:1.36 --restart=Never -- \
  wget -qO- http://web.prod.svc.cluster.local      # page "environnement prod"
```

Pour isoler le réseau, il faut des **NetworkPolicy**, appliquées par un CNI compatible (Calico, Cilium…).

**2. Pourquoi ne jamais committer un Secret en clair dans Git ?**
Le base64 se décode immédiatement. L'historique Git est permanent (anciens commits, clones, forks). Et des robots scannent les dépôts publics à la recherche d'identifiants.
Solutions :
- **Sealed Secrets** : le Secret est chiffré avec la clé publique du contrôleur ; seul le cluster peut le déchiffrer.
- **External Secrets Operator** : la valeur reste dans un coffre externe (Vault, AWS Secrets Manager…), et Git ne contient qu'une référence.

**3. Deployment vs StatefulSet**

| Critère | Deployment | StatefulSet |
| --- | --- | --- |
| Nom des Pods | Aléatoire (`web-6b6f964cd8-9rs9f`) | Stable et ordonné (`postgres-0`, `postgres-1`…) |
| Ordre de démarrage | En parallèle | Un par un (`-0` puis `-1`), arrêt en ordre inverse |
| Volumes | PVC éventuel partagé | Un PVC par Pod (`volumeClaimTemplates`) |
| Réseau | Service avec IP virtuelle | Service headless, un nom DNS par Pod |

**4. Que deviennent le PVC et les données si l'on supprime le StatefulSet ?**
Ils sont **conservés**. Vérifié avec les commandes suivantes :

```bash
kubectl delete statefulset postgres
kubectl get pods,pvc                # postgres-0 Terminating, data-postgres-0 toujours Bound
kubectl apply -f postgres.yaml
kubectl exec -it postgres-0 -- psql -U etudiant -d ensah -c "SELECT * FROM etudiants;"   # Amine, Salma
```

Le PVC doit être supprimé à la main (`kubectl delete pvc data-postgres-0`). Avec la politique `Delete` de la StorageClass, le PV et les données sont alors détruits.

---

## Défi — ResourceQuota

```bash
kubectl apply -f dev-quota.yaml
kubectl describe resourcequota dev-quota     # pods : 3 utilisés / 4
kubectl scale deployment web --replicas=6
kubectl get rs                               # web-6b6f964cd8 : DESIRED 6, CURRENT 3
kubectl describe rs web-6b6f964cd8
```

Message obtenu dans les **Events** du ReplicaSet :

```text
Warning  FailedCreate  replicaset-controller  Error creating: pods "web-6b6f964cd8-64kr2" is forbidden:
exceeded quota: dev-quota, requested: pods=1, used: pods=4, limited: pods=4
```

Le quota compte tous les Pods de `dev` (2 `web` + `postgres-0`). Un seul Pod supplémentaire a donc pu être créé. Retour à la normale : `kubectl scale deployment web --replicas=2`.

---

