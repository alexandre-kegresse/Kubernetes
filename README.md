# Kubernetes — Cluster K3S

Mise en place d'un cluster Kubernetes léger avec K3S sur 3 VMs Debian.

## Infrastructure

| Hostname | IP | Rôle |
|---|---|---|
| kubes-01.local | 192.168.58.155 | Master (control-plane) |
| kubes-02.local | 192.168.58.156 | Worker |
| kubes-03.local | 192.168.58.157 | Worker |

**OS :** Debian (sans GUI)  
**K3S :** v1.35.5+k3s1  
**Hyperviseur :** VMware

---

## Job 01 — Installation de K3S sur chaque VM

K3S est une distribution Kubernetes légère, idéale pour les environnements avec peu de ressources. On installe d'abord K3S en mode standalone sur chaque VM, avant de les regrouper en cluster.

### Configuration des hostnames

```bash
# Sur kubes-01
sudo hostnamectl set-hostname kubes-01.local

# Sur kubes-02
sudo hostnamectl set-hostname kubes-02.local

# Sur kubes-03
sudo hostnamectl set-hostname kubes-03.local
```

### Configuration de /etc/hosts (sur les 3 VMs)

```
192.168.58.155   kubes-01.local
192.168.58.156   kubes-02.local
192.168.58.157   kubes-03.local
```

### Installation de K3S

```bash
curl -sfL https://get.k3s.io | sh -
```

### Vérification

```bash
k3s kubectl get nodes
```

Résultat attendu sur chaque VM :
```
NAME             STATUS   ROLES           AGE   VERSION
kubes-0X.local   Ready    control-plane   Xs    v1.35.5+k3s1
```

---

## Job 02 — Déploiement des applications (standalone)

> Déploiement de nginx, apache et mariadb sur chaque VM en mode standalone, avant la mise en cluster.

### Déployer nginx

```bash
k3s kubectl create deployment nginx --image=nginx
k3s kubectl expose deployment nginx --port=80 --type=NodePort
```

### Déployer Apache

```bash
k3s kubectl create deployment apache --image=httpd
k3s kubectl expose deployment apache --port=80 --type=NodePort
```

### Déployer MariaDB

```bash
k3s kubectl create deployment mariadb --image=mariadb \
  --env="MARIADB_ROOT_PASSWORD=Secret123"
k3s kubectl expose deployment mariadb --port=3306 --type=ClusterIP
```

### Vérification

```bash
k3s kubectl get deployments
k3s kubectl get pods
k3s kubectl get services
```

---

## Job 03 — Création du cluster K3S

> kubes-01 devient le master, kubes-02 et kubes-03 rejoignent en tant que workers.

### Récupérer le token du master (sur kubes-01)

```bash
cat /var/lib/rancher/k3s/server/node-token
```

### Désinstaller K3S standalone (sur kubes-02 et kubes-03)

Les workers tournaient en mode serveur standalone, il faut d'abord les nettoyer :

```bash
/usr/local/bin/k3s-uninstall.sh
```

### Rejoindre le cluster (sur kubes-02 et kubes-03)

> Le token contient des `::`, il faut donc l'entourer de guillemets pour éviter que bash l'interprète.

```bash
curl -sfL https://get.k3s.io | K3S_URL=https://192.168.58.155:6443 \
  K3S_TOKEN="<TOKEN>" sh -
```

K3S s'installe alors en mode **agent** (worker) au lieu de serveur.

### Vérification depuis le master

```bash
k3s kubectl get nodes
```

Résultat :
```
NAME             STATUS   ROLES           VERSION
kubes-01.local   Ready    control-plane   v1.35.5+k3s1
kubes-02.local   Ready    <none>          v1.35.5+k3s1
kubes-03.local   Ready    <none>          v1.35.5+k3s1
```

`<none>` dans la colonne ROLES = nœud worker. Le cluster est formé.

### Vérification des applications

```bash
k3s kubectl get pods -o wide   # -o wide montre le nœud où tourne chaque pod
k3s kubectl get services
```

Les pods nginx/apache/mariadb créés au Job 02 sur le master sont toujours là et accessibles depuis tout le cluster.

---

## Job 04 — Haute Disponibilité (HA)

> Réinstallation des applications avec des replicas pour garantir la disponibilité en cas de panne d'un worker.

### Déploiement avec replicas

```bash
k3s kubectl create deployment nginx --image=nginx --replicas=3
k3s kubectl create deployment apache --image=httpd --replicas=3
k3s kubectl create deployment mariadb --image=mariadb --replicas=2 \
  --env="MARIADB_ROOT_PASSWORD=Secret123"
```

> **Note sur MariaDB :** on utilise `kubectl set env` après le `create deployment` car
> `create deployment` n'accepte pas le flag `--env`. Avec 2 replicas sur 3 nœuds, un nœud
> reste sans pod mariadb — c'est normal. En production, répliquer une base SQL demande une
> vraie config de réplication (master/slave, Galera), pas de simples replicas.

### Test de la HA

On simule la panne d'un worker et on vérifie que Kubernetes redéploie les pods.

```bash
# Sur kubes-03 : arrêter l'agent K3S (simuler une panne)
systemctl stop k3s-agent
```

Depuis le master, le nœud passe d'abord en `NotReady` :

```bash
k3s kubectl get nodes
# kubes-03.local   NotReady   <none>
```

K3S applique un **délai de tolérance de 5 minutes** (300s) avant de considérer un nœud
comme mort, pour éviter de réagir à une micro-coupure réseau. Passé ce délai :

```bash
k3s kubectl get pods -o wide
```

Résultat observé :
- Les pods de kubes-03 passent en `Terminating`
- 3 **nouveaux pods** sont recréés automatiquement sur kubes-01 et kubes-02
- Les compteurs de replicas (3 nginx, 3 apache, 2 mariadb) sont maintenus

**→ La haute disponibilité est démontrée : aucune intervention manuelle.**

### Remettre le worker en service

```bash
# Sur kubes-03
systemctl start k3s-agent
```

Le nœud redevient `Ready`. Les pods existants ne sont pas déplacés, mais kubes-03
redevient disponible pour les futurs déploiements.

---

## Job 05 — Volumes persistants

> Les volumes permettent de conserver les données même si un pod redémarre ou est recréé.

### Le principe

Quand un pod est détruit/recréé, ses données internes disparaissent. Un **volume persistant**
stocke les données en dehors du pod, sur le disque du nœud.

| Terme | Rôle |
|---|---|
| **PV** (PersistentVolume) | Le stockage physique réel |
| **PVC** (PersistentVolumeClaim) | La demande de stockage faite par une application |
| **StorageClass** | Le provisionneur qui crée les PV automatiquement |

K3S fournit une StorageClass par défaut `local-path` (provisioner `rancher.io/local-path`)
qui crée les PV automatiquement, en mode `WaitForFirstConsumer` (le volume est créé sur le
nœud où le pod est planifié).

> **Compromis HA vs stockage local :** `local-path` est en `ReadWriteOnce` (un seul nœud
> peut le monter). On ne peut donc pas avoir 3 replicas multi-nœuds partageant le même volume.
> nginx et mariadb repassent à **1 replica** pour le stockage persistant. Une vraie HA avec
> stockage partagé nécessiterait du NFS, Longhorn ou un stockage réseau.

### Vérifier la StorageClass

```bash
k3s kubectl get storageclass
# local-path (default)   rancher.io/local-path   Delete   WaitForFirstConsumer
```

### Manifestes

Voir `manifests/05-nginx-pv.yaml` et `manifests/05-mariadb-pv.yaml`.
Chacun définit un `PersistentVolumeClaim` (1Gi, RWO) et un `Deployment` qui monte ce PVC :
- nginx : volume monté sur `/usr/share/nginx/html`
- mariadb : volume monté sur `/var/lib/mysql`

### Déploiement

```bash
k3s kubectl delete deployment nginx mariadb   # supprimer les anciens
k3s kubectl apply -f 05-nginx-pv.yaml
k3s kubectl apply -f 05-mariadb-pv.yaml
k3s kubectl get pvc                            # les PVC doivent être "Bound"
```

### Test de persistance

```bash
# Écrire un fichier dans le volume nginx
k3s kubectl exec <pod-nginx> -- sh -c "echo 'Donnee persistante' > /usr/share/nginx/html/test.html"

# Détruire le pod (Kubernetes en recrée un automatiquement)
k3s kubectl delete pod <pod-nginx>

# Lire le fichier dans le NOUVEAU pod : la donnée a survécu
k3s kubectl exec <nouveau-pod-nginx> -- cat /usr/share/nginx/html/test.html
# -> Donnee persistante
```

**→ Le fichier survit à la destruction du pod : le stockage persistant fonctionne.**

---

## Job 06 — ConfigMaps

> Gérer la configuration des applications de façon externalisée.

### Le principe

Un **ConfigMap** externalise la configuration hors de l'image Docker. Au lieu de reconstruire
une image pour changer un paramètre, on modifie le ConfigMap. La config peut être injectée
comme variable d'environnement ou comme fichier monté dans le pod.

### Manifeste

Voir `manifests/06-nginx-configmap.yaml`. Il contient :
- un `ConfigMap` `nginx-config` avec une config nginx personnalisée (`default.conf`)
- le `Deployment` nginx mis à jour pour monter ce ConfigMap dans `/etc/nginx/conf.d`

La config fait répondre nginx avec un message personnalisé.

### Déploiement

```bash
k3s kubectl apply -f 06-nginx-configmap.yaml
k3s kubectl get configmap        # nginx-config apparaît
```

### Test

```bash
k3s kubectl get service nginx    # récupérer le NodePort (ex: 80:31390)
curl http://localhost:31390
# -> Configuration geree par ConfigMap - Runtrack K8S

# Vérifier le fichier monté dans le conteneur
k3s kubectl exec <pod-nginx> -- cat /etc/nginx/conf.d/default.conf
```

**→ nginx sert la configuration injectée par le ConfigMap : la config est bien externalisée.**

---

## Job 07 — Secrets

> Stocker les données sensibles (mots de passe, clés) de façon sécurisée pour MariaDB.

### Le principe

Un **Secret** est comme un ConfigMap mais pour les données sensibles. Les valeurs sont
encodées en base64 et Kubernetes en restreint l'accès (jamais affichées en clair).
On remplace le mot de passe root MariaDB écrit en clair par une référence à un Secret.

### Créer le Secret

```bash
k3s kubectl create secret generic mariadb-secret \
  --from-literal=MARIADB_ROOT_PASSWORD=Secret123

k3s kubectl describe secret mariadb-secret
# MARIADB_ROOT_PASSWORD:  9 bytes   <- valeur jamais affichée
```

### Manifeste

Voir `manifests/07-mariadb-secret.yaml`. Le mot de passe passe de :

```yaml
env:
  - name: MARIADB_ROOT_PASSWORD
    value: Secret123          # en clair (avant)
```

à :

```yaml
env:
  - name: MARIADB_ROOT_PASSWORD
    valueFrom:
      secretKeyRef:           # lu depuis le Secret (après)
        name: mariadb-secret
        key: MARIADB_ROOT_PASSWORD
```

### Déploiement et test

```bash
k3s kubectl apply -f 07-mariadb-secret.yaml
k3s kubectl exec <pod-mariadb> -- mariadb -uroot -pSecret123 -e "SHOW DATABASES;"
# -> liste des bases : MariaDB utilise le mot de passe injecté depuis le Secret
```

**→ Le mot de passe n'est plus en clair dans le manifeste : il provient du Secret.**

---

## Job 08 — RBAC

> Contrôle d'accès basé sur les rôles (Role-Based Access Control).

### Le principe

Le RBAC contrôle **qui** peut faire **quoi** sur le cluster (principe du moindre privilège).

| Objet | Rôle |
|---|---|
| **ServiceAccount** | Une identité (compte pour une appli ou personne) |
| **Role** | Un ensemble de permissions dans un namespace |
| **RoleBinding** | Attribue un Role à un ServiceAccount |
| **ClusterRole / ClusterRoleBinding** | Idem mais à l'échelle du cluster entier |

On crée un compte `lecteur` autorisé à lire les pods (get/list/watch) mais
pas à les modifier/supprimer.

### Manifeste

Voir `manifests/08-rbac.yaml` : un `ServiceAccount` (lecteur), un `Role`
(lecteur-pods, verbes get/list/watch sur pods) et un `RoleBinding` qui relie les deux.

### Déploiement et test

```bash
k3s kubectl apply -f 08-rbac.yaml

k3s kubectl auth can-i list pods   --as=system:serviceaccount:default:lecteur   # yes
k3s kubectl auth can-i delete pods --as=system:serviceaccount:default:lecteur   # no
k3s kubectl auth can-i create deployments --as=system:serviceaccount:default:lecteur  # no
```

**→ Le compte `lecteur` peut lire mais pas modifier : le RBAC applique le moindre privilège.**

---

## Job 09 — Helm

> Gestionnaire de packages pour Kubernetes (l'équivalent d'`apt` pour K8S).

### Le principe

Helm installe des applications complètes via des **charts** (packages préconfigurés),
en une commande au lieu de multiples fichiers YAML. Il gère aussi un historique de
révisions (upgrade / rollback).

### Installation de Helm

```bash
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml   # indiquer la config K3S à Helm
helm version
```

### Cycle complet : rechercher → installer → personnaliser → mettre à jour → désinstaller

```bash
# Ajouter un dépôt de charts et le rafraîchir
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Rechercher
helm search repo nginx

# Installer avec personnalisation (2 replicas, service NodePort)
helm install mon-nginx bitnami/nginx --set replicaCount=2 --set service.type=NodePort
helm list                 # REVISION 1

# Mettre à jour (passer à 3 replicas)
helm upgrade mon-nginx bitnami/nginx --set replicaCount=3 --set service.type=NodePort
helm list                 # REVISION 2

# Désinstaller
helm uninstall mon-nginx
helm list                 # vide
```

> **Note Bitnami :** depuis le 28/08/2025, Bitnami ne propose qu'un sous-ensemble limité
> d'images gratuites. Le chart reste fonctionnel pour la démonstration.

**→ Cycle Helm complet démontré : un chart gère tout le déploiement, versionné et réversible.**

---

## Conclusion

Les 9 jobs ont permis de monter un cluster K3S complet (1 master + 2 workers) et de couvrir
les concepts fondamentaux de Kubernetes :

| Job | Concept | Statut |
|---|---|---|
| 01 | Installation K3S | ✅ |
| 02 | Déploiements (Pods, Services) | ✅ |
| 03 | Cluster (master + workers) | ✅ |
| 04 | Haute Disponibilité (replicas) | ✅ |
| 05 | Volumes persistants (PV/PVC) | ✅ |
| 06 | ConfigMaps | ✅ |
| 07 | Secrets | ✅ |
| 08 | RBAC | ✅ |
| 09 | Helm | ✅ |

---

## Comparaison K3S vs Docker vs Docker Swarm

| Critère | Docker | Docker Swarm | K3S |
|---|---|---|---|
| Orchestration | Non | Oui (basique) | Oui (Kubernetes complet) |
| Complexité | Faible | Moyenne | Moyenne/Haute |
| Ressources | Faibles | Moyennes | Faibles (optimisé) |
| Scalabilité | Non | Limitée | Haute |
| Écosystème | Large | Moyen | Très large (Kubernetes) |
| Cas d'usage | Dev local | Petits clusters | Prod, edge, IoT |
