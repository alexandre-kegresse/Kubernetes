# Fiche de soutenance — Cluster K3S

Fiche anti-sèche pour la démo live. Ordre de passage, commandes prêtes, et points à dire à l'oral.

---

## Préparation avant de commencer (à faire 5 min avant)

- [ ] Les 3 VMs sont allumées (kubes-01, kubes-02, kubes-03)
- [ ] 3 terminaux ouverts (un par VM), ou au minimum le master (kubes-01)
- [ ] Sur kubes-01 : `export KUBECONFIG=/etc/rancher/k3s/k3s.yaml` (pour Helm)
- [ ] Vérifier que tout tourne : `k3s kubectl get nodes` (3x Ready)
- [ ] Avoir les captures d'écran de secours sous la main (au cas où une démo plante)

---

## Pitch d'intro (30 secondes)

> "J'ai monté un cluster Kubernetes léger avec K3S sur 3 VMs Debian : un master et deux
> workers. K3S c'est une distribution Kubernetes allégée, parfaite pour les petites
> infrastructures et l'edge. Je vais vous montrer le déploiement d'applications, la haute
> disponibilité, le stockage persistant, la gestion de configuration et des secrets, le
> contrôle d'accès, et enfin Helm pour packager des applications."

---

## Déroulé de la démo (ordre conseillé)

### 1. Le cluster (Jobs 01 + 03)

```bash
k3s kubectl get nodes
```

**À dire :** "Voici mes 3 nœuds. kubes-01 est le control-plane (le master, le cerveau du
cluster), kubes-02 et kubes-03 sont les workers qui exécutent les applications. Le master
décide où placer les pods, les workers les font tourner."

---

### 2. Les applications déployées (Job 02)

```bash
k3s kubectl get pods -o wide
k3s kubectl get services
```

**À dire :** "J'ai déployé nginx, apache et mariadb. Un *pod* c'est l'unité de base, il
contient le conteneur. Un *service* expose le pod sur le réseau. La colonne NODE montre sur
quel worker chaque pod tourne — c'est le scheduler de Kubernetes qui répartit la charge."

---

### 3. La Haute Disponibilité (Job 04) — LE MOMENT FORT

```bash
# Montrer les replicas répartis
k3s kubectl get pods -o wide

# Simuler une panne sur kubes-03
# (sur le terminal kubes-03)
systemctl stop k3s-agent

# Revenir sur kubes-01, montrer le nœud NotReady
k3s kubectl get nodes

# Après ~5 min : les pods sont recréés ailleurs
k3s kubectl get pods -o wide
```

**À dire :** "Avec plusieurs replicas, si un worker tombe, Kubernetes redéploie
automatiquement les pods sur les nœuds sains, sans intervention. C'est ça la haute
disponibilité. K3S attend 5 minutes avant de réagir, pour ne pas paniquer sur une simple
micro-coupure réseau."

> ⚠️ **Astuce démo :** la HA prend 5 min. Si tu n'as pas le temps en live, montre tes
> captures d'écran avant/après. Relance `systemctl start k3s-agent` sur kubes-03 après.

---

### 4. Volumes persistants (Job 05)

```bash
# Écrire une donnée
k3s kubectl exec <pod-nginx> -- sh -c "echo 'test' > /usr/share/nginx/html/test.html"
# Détruire le pod
k3s kubectl delete pod <pod-nginx>
# Lire dans le nouveau pod : la donnée a survécu
k3s kubectl exec <nouveau-pod-nginx> -- cat /usr/share/nginx/html/test.html
```

**À dire :** "Sans volume, détruire un pod efface ses données. Avec un PersistentVolumeClaim,
les données sont stockées sur le disque du nœud et survivent. Crucial pour une base de
données."

---

### 5. ConfigMap (Job 06)

```bash
curl http://localhost:<nodeport-nginx>
```

**À dire :** "nginx renvoie un message que j'ai défini dans un ConfigMap. Le ConfigMap
externalise la configuration : je change un paramètre sans reconstruire l'image Docker."

---

### 6. Secret (Job 07)

```bash
k3s kubectl describe secret mariadb-secret   # montre "9 bytes", pas la valeur
k3s kubectl exec <pod-mariadb> -- mariadb -uroot -pSecret123 -e "SHOW DATABASES;"
```

**À dire :** "Le mot de passe de MariaDB n'est plus écrit en clair dans mon YAML, il est
dans un Secret. Kubernetes ne l'affiche jamais en clair. MariaDB le récupère au démarrage."

---

### 7. RBAC (Job 08)

```bash
k3s kubectl auth can-i list pods   --as=system:serviceaccount:default:lecteur   # yes
k3s kubectl auth can-i delete pods --as=system:serviceaccount:default:lecteur   # no
```

**À dire :** "J'ai créé un compte 'lecteur' qui peut voir les pods mais pas les supprimer.
C'est le principe du moindre privilège : chaque identité n'a que les droits dont elle a
besoin."

---

### 8. Helm (Job 09)

```bash
helm install mon-nginx bitnami/nginx --set replicaCount=2 --set service.type=NodePort
helm list
helm upgrade mon-nginx bitnami/nginx --set replicaCount=3 --set service.type=NodePort
helm list                  # REVISION 2
helm uninstall mon-nginx
```

**À dire :** "Helm c'est le gestionnaire de paquets de Kubernetes, comme apt. J'installe une
appli complète en une commande, je la personnalise, je la mets à jour avec un historique de
versions, et je la désinstalle proprement."

---

## Pour aller plus loin : K3S vs Docker vs Docker Swarm

| Critère | Docker | Docker Swarm | K3S (Kubernetes) |
|---|---|---|---|
| Rôle | Exécuter des conteneurs | Orchestrateur simple | Orchestrateur complet |
| Mise en place | Très simple | Simple | Moyenne |
| Haute dispo | Non | Oui (basique) | Oui (avancée, self-healing) |
| Scalabilité | Non | Limitée | Très élevée |
| Écosystème | Large | Réduit, en déclin | Énorme (standard du marché) |
| Quand l'utiliser | Dev local, 1 machine | Petit cluster simple | Production, edge, IoT, multi-nœuds |

**À dire :** "Docker seul ne fait pas d'orchestration. Docker Swarm en fait, simplement, mais
il est en perte de vitesse. K3S apporte toute la puissance de Kubernetes (self-healing,
scalabilité, immense écosystème) en restant léger. On choisit Swarm pour la simplicité sur un
petit projet, et K3S/Kubernetes dès qu'on veut du sérieux ou de la production."

---

## Questions probables du jury (et réponses)

**Q : Différence entre un pod et un conteneur ?**
> Un pod est l'unité de déploiement de Kubernetes. Il contient un ou plusieurs conteneurs qui
> partagent le réseau et le stockage. En général : 1 pod = 1 conteneur.

**Q : Différence master / worker ?**
> Le master (control-plane) gère le cluster : il décide où placer les pods, surveille l'état.
> Les workers exécutent réellement les conteneurs.

**Q : Pourquoi K3S et pas Kubernetes classique ?**
> K3S est une distribution Kubernetes allégée (un seul binaire, ~50 Mo, base de données SQLite
> au lieu d'etcd par défaut). Idéal pour les VMs modestes et l'apprentissage, tout en restant
> 100% compatible Kubernetes.

**Q : Différence ConfigMap / Secret ?**
> Les deux externalisent des données. Le ConfigMap pour la config non sensible, le Secret pour
> les données sensibles (encodées en base64, accès restreint, jamais affichées en clair).

**Q : Comment Kubernetes sait qu'un nœud est tombé ?**
> Le master perd le contact (heartbeat) avec le nœud. Après un délai de tolérance (5 min par
> défaut), il considère ses pods comme morts et les recrée ailleurs.

**Q : NodePort vs ClusterIP ?**
> ClusterIP = service accessible uniquement à l'intérieur du cluster (ex: la base de données).
> NodePort = service accessible depuis l'extérieur via un port sur chaque nœud (ex: nginx).

**Q : Limite de votre HA sur MariaDB ?**
> Bonne question : répliquer une base SQL avec de simples replicas ne suffit pas, les copies
> doivent synchroniser leurs données. En production il faut une vraie réplication
> (master/slave, Galera). Mon stockage local-path est aussi en ReadWriteOnce, donc lié à un
> seul nœud. Une vraie HA de données demanderait du stockage réseau (NFS, Longhorn).

---

## En cas de pépin pendant la démo

- Un pod ne démarre pas → `k3s kubectl describe pod <nom>` puis regarder la section *Events*
- Voir les logs d'un conteneur → `k3s kubectl logs <nom-pod>`
- Une commande échoue → garder son calme, montrer la capture d'écran correspondante
- Helm ne se connecte pas → vérifier `export KUBECONFIG=/etc/rancher/k3s/k3s.yaml`
