# Rendu — Atelier Kubernetes

> Complétez ce fichier au fil de l'atelier, puis faites un commit et un push (voir la section « Rendu » du README).

**Nom / Prénom :**
**Groupe :**

---

## Exercice 1 — Cluster

1. Combien de nœuds affiche `kubectl get nodes` ? Quel est le rôle de chacun ?
   > `kubectl get nodes` affiche **3 nœuds** : 1 control plane (master) et 2 workers
   > (par exemple `k3d-<cluster>-server-0`, `k3d-<cluster>-agent-0`, `k3d-<cluster>-agent-1`).
   
   > - **Control plane (master)** : c'est le « cerveau » du cluster. Il héberge l'API Server
   >   (point d'entrée de `kubectl`), le scheduler (qui décide sur quel nœud placer chaque pod),
   >   les controllers (qui maintiennent l'état désiré, par exemple le bon nombre de réplicas)
   >   et etcd (la base qui stocke l'état du cluster). Il ne fait en principe pas tourner
   >   les applications.
   > - **Workers (agents)** : ce sont les nœuds qui exécutent réellement les applications.
   >   Ils font tourner les pods (via le kubelet et le runtime de conteneurs) et gèrent le
   >   réseau (kube-proxy). Le control plane leur répartit les pods.

2. Que montre `docker ps` ? Qu'est-ce qu'un « nœud » dans k3d, concrètement ?
   >`docker ps` affiche les conteneurs Docker en cours d'exécution dans le Codespace :
   > un conteneur par nœud du cluster (`k3d-...-server-0`, `k3d-...-agent-0`,
   > `k3d-...-agent-1`), plus le conteneur `k3d-...-serverlb` (load balancer qui expose
   > l'API Server) et éventuellement le conteneur `k3d-tools`.
   > Dans k3d, un « nœud » Kubernetes n'est donc pas une machine physique ni une VM :
   > c'est **un conteneur Docker** qui fait tourner K3s (une distribution légère de
   > Kubernetes). Les pods de nos applications tournent à l'intérieur de ces conteneurs-nœuds.
   > C'est ce qui permet de simuler un cluster multi-nœuds sur une seule machine.

## Exercice 2 — Premier déploiement

3. Quels objets Kubernetes ont été créés par `kubectl create deployment` ? (indice : `kubectl get all`)
   >`kubectl create deployment` crée **3 objets** en cascade :
   > - un **Deployment** (`deployment.apps/web`) : il décrit l'état désiré (image, nombre
   >   de réplicas) et gère les mises à jour et les rollbacks ;
   > - un **ReplicaSet** (`replicaset.apps/web-<hash>`) : créé automatiquement par le
   >   Deployment, il s'assure que le bon nombre de pods tourne en permanence ;
   > - un ou plusieurs **Pods** (`pod/web-<hash>-<id>`) : l'unité d'exécution qui contient
   >   le conteneur de l'application, placé sur un worker.
4. Vous avez supprimé le pod `web-...` : que s'est-il passé, et pourquoi ?
   > Après la suppression, un **nouveau pod `web-...` a été recréé automatiquement**,
   > avec un nom différent (suffixe aléatoire) et un nouvel âge (`AGE` de quelques secondes).
   
   > Pourquoi : le **ReplicaSet** compare en permanence l'état réel (0 pod) à l'état désiré
   > défini dans le Deployment (par ex. 1 réplica). Il détecte l'écart et demande la création
   > d'un nouveau pod : c'est l'**auto-réparation** (self-healing) de Kubernetes, basée sur
   > une boucle de réconciliation.
   > Cela montre aussi le caractère **éphémère** des pods : on ne dépend jamais d'un pod
   > précis (son nom et son IP changent), c'est le Deployment qui garantit que
   > l'application reste disponible.

## Exercice 3 — Modifier un conteneur en direct

5. Après suppression du pod, votre message « Bienvenue... » a-t-il survécu ? Pourquoi ?
   >**Non**, le message a disparu. Le message avait été écrit à l'intérieur du conteneur
   > du pod (dans son système de fichiers). Or le système de fichiers d'un conteneur est
   > **éphémère** : il est créé à partir de l'image et détruit avec le conteneur.
   > Quand on supprime le pod, le ReplicaSet en recrée un nouveau, mais à partir de
   > l'image d'origine : il repart donc d'un état vierge, sans nos modifications.
   > Pour conserver des données au-delà de la vie d'un pod, il faut utiliser un
   > **volume persistant** (PersistentVolume / PersistentVolumeClaim), ou stocker
   > les données hors du pod (base de données, stockage externe).
6. Pourquoi le `port-forward` a-t-il cessé de fonctionner après la suppression du pod ?
   >`kubectl port-forward deployment/web ...` (ou `pod/web-...`) ouvre un tunnel vers
   > **un pod précis**, choisi au moment où la commande est lancée. Quand ce pod est
   > supprimé, le tunnel perd sa cible et la connexion se coupe, alors que le nouveau
   > pod a un autre nom et une autre IP : le port-forward ne bascule pas automatiquement
   > dessus.
   >
   > Il faut donc relancer la commande pour recréer le tunnel vers le nouveau pod.
   > C'est une des limites du port-forward, qui sert au débogage. Pour un accès stable
    malgré le renouvellement des pods, on utilise un **Service**, qui fournit une adresse
   > fixe et répartit le trafic vers les pods disponibles, éventuellement exposé via
    un **Ingress**.

## Exercice 4 — Déclaratif

7. Quelle différence entre `kubectl create` et `kubectl apply` ?
   >`kubectl create` est **impératif** : on dit à Kubernetes « crée cet objet maintenant ».
   > Si l'objet existe déjà, la commande échoue (`AlreadyExists`). Elle ne sert qu'à créer.
   > `kubectl apply` est **déclaratif** : on fournit l'état désiré (un fichier YAML) et
   > Kubernetes s'occupe d'y parvenir. Si l'objet n'existe pas, il est créé ; s'il existe
   > déjà, il est mis à jour avec les différences. On peut donc relancer `apply` autant de
   > fois qu'on veut (c'est idempotent) : la sortie indique `created`, `configured` ou `unchanged`.
   > En pratique, `create` convient pour un test rapide (`kubectl create deployment web ...`),
   > alors que `apply -f` est la méthode « production » : les manifestes sont versionnés
   > dans Git, relisibles, reproductibles, et on modifie le fichier plutôt que de relancer des commandes à la main.
8. À quoi sert le `selector` d'un Service ? Que se passe-t-il s'il ne correspond à aucun pod ?
   > Le `selector` indique **quels pods reçoivent le trafic** du Service : le Service
   > cible tous les pods dont les labels correspondent (ici `app: site`). C'est ce qui
   > permet d'avoir une adresse stable (nom DNS et IP du Service) alors que les pods, eux, sont éphémères et changent de nom et d'IP : le Service suit automatiquement les pods
   > qui portent le bon label et répartit le trafic entre eux.
   > Si le selector ne correspond à **aucun pod**, le Service existe toujours mais n'a
   > **aucun endpoint** (liste vide dans `kubectl get endpointslices`). Le trafic n'est
   > envoyé nulle part : le `port-forward svc/site` échoue ou la connexion est refusée, et
   > l'application est inaccessible, alors que les pods tournent normalement. La cause
   > classique est une faute de frappe dans un label, d'où l'importance que `app: site`
   > soit identique dans le Service, dans le `selector` du Deployment et dans les labels
   > du template.

## Exercice 5 — Scaling & mises à jour

9. Sur quels nœuds se trouvent vos 3 réplicas ? Collez la sortie de `kubectl get pods -l app=site -o wide`.
   ```
Les 3 réplicas sont répartis sur les **workers** (`agent-0` et `agent-1`) par le
   > scheduler, qui équilibre la charge. Aucun pod ne tourne sur le control plane.
   ```
10. Avec l'image `nginx:9.99`, quel statut avait le pod ? Quelle commande vous a donné la cause ?
    > Le nouveau pod était en **`ErrImagePull`**, puis **`ImagePullBackOff`** : Kubernetes
    > n'a pas réussi à télécharger l'image `nginx:9.99`, qui n'existe pas, et il
    > réessaie avec des délais de plus en plus longs.
    > La cause est visible avec **`kubectl describe pod <nom-du-pod>`** (ou `kubectl describe pod -l app=site`), dans la section **Events** en bas :
    > `Failed to pull image "nginx:9.99": ... not found`.
    > `kubectl get pods` indique seulement le statut, pas la raison.
11. Pendant la panne, le site restait-il accessible ? Pourquoi ?
    > **Oui**, le site est resté accessible. Lors d'une mise à jour, le Deployment fait un
    > **rolling update** : il crée le nouveau pod avant de supprimer les anciens, et il ne
    > continue que si le nouveau pod est `Ready`. Ici, le nouveau pod n'est jamais devenu
    > prêt, donc le Deployment a **conservé les anciens pods** (avec l'ancienne image
    > `nginx:1.27`) qui continuaient à servir les requêtes.
    > De plus, le Service n'envoie le trafic qu'aux pods **Ready** : le pod en erreur ne
    > recevait aucune requête. Il n'y a donc eu aucune interruption de service, et on peut
    > revenir en arrière avec `kubectl rollout undo deploy/site`.

## Exercice 6 — Portainer & Mario

12. URL publique de votre Mario :
    > https://reimagined-space-umbrella-g44r994rprpx29wvp-8585.app.github.dev/
13. Retrouvez le Deployment `mario` en ligne de commande. Portainer et `kubectl` parlent-ils au même composant ? Lequel ?
    >NAME    READY   UP-TO-DATE   AVAILABLE   AGE
mario   1/1     1            1           5m21s

Le Deployment `mario`, créé via l'interface Portainer, apparaît bien en ligne de commande
   > (1/1 pod prêt et disponible) : il est donc dans le cluster.
   >
   > **Portainer et `kubectl` parlent au même composant : l'API Server** du control plane.
   > Portainer est seulement une interface graphique qui envoie ses requêtes à l'API Server,
   > comme `kubectl`. L'état est stocké dans etcd, et le scheduler et les controllers créent
   > les pods sur les workers. Il n'y a qu'une seule source de vérité, quel que soit l'outil.

## Bonus (facultatif)

14. Bonus réalisés et remarques :
    >

---

**Score obtenu avec `./scripts/check.sh` :** ... / ...
