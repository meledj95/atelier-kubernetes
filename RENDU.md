# Rendu — Atelier Kubernetes

> Complétez ce fichier au fil de l'atelier, puis faites un commit et un push (voir la section « Rendu » du README).

**Nom / Prénom :**
**Groupe :**

---

## Exercice 1 — Cluster

1. Combien de nœuds affiche `kubectl get nodes` ? Quel est le rôle de chacun ?
   > k3d-lab-server-0 est le nœud du plan de contrôle. Il héberge l’API server, le scheduler, le controller manager et le datastore. C’est lui qui décide où placer les pods et qui maintient l’état souhaité du cluster.  k3d-lab-agent-0 et k3d-lab-agent-1 sont les nœuds de travail (workers). Ils exécutent les pods applicatifs. La colonne ROLES affiche <none> car K3s ne leur attribue pas de label de rôle worker par défaut. Cela ne signifie pas qu’ils sont inactifs.
2. Que montre `docker ps` ? Qu'est-ce qu'un « nœud » dans k3d, concrètement ?
   > kubectl get nodes affiche 3 nœuds Ready : 1 control-plane (k3d-lab-server-0) et 2 workers (k3d-lab-agent-0 et agent-1).
Le control-plane gère le cluster (API server, scheduler, état souhaité), les workers exécutent les pods.
docker ps montre 4 conteneurs : les 3 nœuds (image rancher/k3s:v1.35.5-k3s1) plus k3d-lab-serverlb.
Dans k3d, un nœud est simplement un conteneur Docker qui fait tourner K3s, sans VM ni machine physique.
serverlb n’est pas un nœud : c’est un proxy qui expose l’API server à votre machine hôte pour que kubectl puisse joindre le cluster.

## Exercice 2 — Premier déploiement

3. Quels objets Kubernetes ont été créés par `kubectl create deployment` ? (indice : `kubectl get all`)
   >kubectl create deployment a créé 3 objets en cascade : un Deployment (web), un ReplicaSet (web-68d995574f) et un Pod (web-68d995574f-ptsnz).
Le Deployment déclare l’état souhaité (image, réplicas) et gère les mises à jour.
Le ReplicaSet garantit que le nombre de pods voulu (1) tourne en permanence, et crée le Pod.
Le service/web n’est pas créé par cette commande : il vient d’un kubectl expose séparé, et service/kubernetes est créé automatiquement par le cluster.
Le pod h2hr8 supprimé puis remplacé par ptsnz illustre l’auto-réparation : le ReplicaSet recrée immédiatement un pod manquant.

4. Vous avez supprimé le pod `web-...` : que s'est-il passé, et pourquoi ?
   >Le pod n2hr8 a été supprimé : Kubernetes l’a arrêté (signal SIGTERM, puis SIGKILL après le délai de grâce).
Le ReplicaSet web-68d995574f a détecté qu’il n’y avait plus que 0 pod pour 1 voulu (DESIRED = 1).
Il a aussitôt créé un nouveau pod, web-68d995574f-ptsnz, visible dans votre kubectl get all avec un âge de 71s.

## Exercice 3 — Modifier un conteneur en direct

5. Après suppression du pod, votre message « Bienvenue... » a-t-il survécu ? Pourquoi ?
   >Non, le message « Bienvenue... » n’a pas survécu : il avait été écrit dans le conteneur du pod, donc dans une couche éphémère qui disparaît avec lui.
Le ReplicaSet a recréé un nouveau pod à partir de l’image d’origine, sans votre modification.
L’erreur 404 vient probablement d’un port-forward resté sur l’ancien pod, d’un index.html absent de l’image, ou de Traefik qui ne trouve pas de route.
Pour diagnostiquer : kubectl get endpoints web et kubectl exec deploy/web -- ls /usr/share/nginx/html.
Pour rendre le message persistant, montez un ConfigMap (ou un volume) sur /usr/share/nginx/html, ou construisez une image personnalisée.
6. Pourquoi le `port-forward` a-t-il cessé de fonctionner après la suppression du pod ?
   >Quand vous lancez kubectl port-forward deployment/web 8080:80 (ou service/web), kubectl résout la cible vers un seul pod à cet instant (ici n2hr8).
Il ouvre un tunnel entre le port local 8080 et ce pod, via l’API server (flux SPDY/WebSocket vers le kubelet du nœud).
Quand vous supprimez ce pod, le tunnel est coupé : sa destination n’existe plus. Vous voyez généralement une erreur du type lost connection to pod ou error forwarding port, et le processus s’arrête ou ne répond plus.
Le ReplicaSet crée un nouveau pod (ptsnz), mais kubectl ne rebascule pas automatiquement : il ne suit pas les remplacements de pods.

## Exercice 4 — Déclaratif

7. Quelle différence entre `kubectl create` et `kubectl apply` ?
   >kubectl create est impératif : il crée une ressource et échoue avec une erreur AlreadyExists si elle existe déjà.
kubectl apply est déclaratif : il crée la ressource si elle n’existe pas, sinon il la met à jour pour qu’elle corresponde au fichier YAML.
apply mémorise la dernière configuration appliquée (annotation last-applied-configuration) et calcule la différence, alors que create ne fait rien de tel.
On peut donc relancer apply -f fichier.yaml autant de fois qu’on veut (idempotent), ce qui est idéal pour modifier un Deployment ou l’intégrer à du GitOps.
En pratique : create pour un essai rapide ou une création unique, apply pour tout ce qui est versionné dans des fichiers YAML et destiné à évoluer.
8. À quoi sert le `selector` d'un Service ? Que se passe-t-il s'il ne correspond à aucun pod ?
   > Le selector d’un Service indique quels pods il cible, en les sélectionnant par leurs labels (par exemple app: web).
Kubernetes maintient automatiquement la liste des pods correspondants (les endpoints) et le Service répartit le trafic entre eux, même quand les pods sont remplacés.
Si le selector ne correspond à aucun pod, le Service est quand même créé et garde son ClusterIP, mais sa liste d’endpoints reste vide.
Résultat : les requêtes vers le Service échouent (connexion refusée ou timeout), ou renvoient une erreur 503/404 si un Ingress/Traefik est devant.
Pour vérifier : kubectl get endpoints web (doit lister des IP) et comparez kubectl describe svc web avec kubectl get pods --show-labels.

## Exercice 5 — Scaling & mises à jour

9. Sur quels nœuds se trouvent vos 3 réplicas ? Collez la sortie de `kubectl get pods -l app=site -o wide`.
   ```
   Mes 3 réplicas Running (hash 5cc664db9c) : Pod	Nœud	IP site-5cc664db9c-mwvhj	k3d-lab-agent-0	10.42.2.7 site-5cc664db9c-sq492	k3d-lab-agent-1	10.42.1.8 site-5cc664db9c-vsdwr	k3d-lab-server-0	10.42.0.7
   ```
10. Avec l'image `nginx:9.99`, quel statut avait le pod ? Quelle commande vous a donné la cause ?
    >Le pod site-fc576997-f7tvn était en ErrImagePull, puis alternait avec ImagePullBackOff (visible dans votre kubectl get pods -w). Il restait à 0/1 et n’est jamais passé Ready, car le tag nginx:9.99 n’existe pas sur Docker Hub. Les deux statuts : ErrImagePull : l’échec du téléchargement de l’image. ImagePullBackOff : Kubernetes attend avant de réessayer, avec un délai croissant (visible dans vos captures : 1s, 16s, 31s, 45s, 60s). La commande qui donne la cause : kubectl describe pod site-fc576997-f7tvn
    
11. Pendant la panne, le site restait-il accessible ? Pourquoi ?
    >Oui, le site restait accessible. C’est exactement ce que le rolling update est censé garantir. Pourquoi : Le Deployment met à jour progressivement : il crée d’abord le nouveau pod (fc576997) avant de supprimer les anciens, selon la stratégie RollingUpdate (par défaut maxSurge: 25% et maxUnavailable: 25%). Le nouveau pod est resté bloqué en ErrImagePull / ImagePullBackOff : il n’est jamais devenu Ready (0/1). Un pod non Ready n’est jamais ajouté aux endpoints du Service, donc aucun trafic ne lui est envoyé. Les 3 anciens pods (5cc664db9c) sont restés 1/1 Running, car le Deployment ne les remplace que lorsqu’un nouveau pod est prêt. Le Service a donc continué à répartir les requêtes entre eux. Ce que ça montre : le rolling update est une protection. Une mauvaise image ne provoque pas de panne, elle bloque simplement le déploiement. Le prix à payer : le rollout reste en attente, ce que vous voyez avec kubectl rollout status deployment site (il reste en « Waiting for rollout to finish »).

## Exercice 6 — Portainer & Mario

12. URL publique de votre Mario :
    >https://animated-chainsaw-jrggr9gx9jg5h5pj7-8585.app.github.dev/
13. Retrouvez le Deployment `mario` en ligne de commande. Portainer et `kubectl` parlent-ils au même composant ? Lequel ?
    >Pour retrouver le Deployment en ligne de commande : kubectl get deploy mario (ou kubectl get deploy mario -o yaml pour le détail).
Il est identique à ceux créés en CLI, car Portainer a simplement envoyé le manifeste mario.yaml au cluster.
Oui, Portainer et kubectl parlent au même composant : l’API Server, sur le nœud control-plane k3d-lab-server-0.
kubectl le contacte depuis votre Codespace via ~/.kube/config (par le load balancer serverlb), alors que Portainer le contacte de l’intérieur du cluster avec un ServiceAccount.
Ce sont deux clients de la même API : un objet créé avec l’un est donc visible avec l’autre.

## Bonus (facultatif)

14. Bonus réalisés et remarques :
    >

---

**Score obtenu avec `./scripts/check.sh` :** ... / ...
