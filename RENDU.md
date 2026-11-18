# Rendu — Atelier Kubernetes

> Complétez ce fichier au fil de l'atelier, puis faites un commit et un push (voir la section « Rendu » du README).

**Nom / Prénom :** DO RIO PEREIRA David
**Groupe :** ING3 RS2

---

## Exercice 1 — Cluster

1. Combien de nœuds affiche `kubectl get nodes` ? Quel est le rôle de chacun ?
   >J'obtiens 3 nœuds, tous en état Ready. Le nœud `k3d-lab-server-0` a le rôle control-plane, c'est le master du cluster. Il fait tourner l'API Server, le scheduler et la base d'état, et c'est lui qui décide sur quel nœud placer chaque pod. Les nœuds `k3d-lab-agent-0` et `k3d-lab-agent-1` sont mes deux workers, ce sont eux qui exécutent les pods des applications. J'ai aussi remarqué que dans k3s le control-plane n'est pas bloqué, il peut donc lui aussi recevoir des pods applicatifs.
2. Que montre `docker ps` ? Qu'est-ce qu'un « nœud » dans k3d, concrètement ?
   >Dans `docker ps` je vois 4 conteneurs. Les trois premiers portent le nom de mes nœuds et utilisent l'image `rancher/k3s`, le quatrième est `k3d-lab-serverlb`, le load balancer qui relie le port 8081 de mon Codespace au port 80 du cluster. J'en conclus que dans k3d un nœud est tout simplement un conteneur Docker qui fait tourner k3s, chaque machine du cluster étant simulée par un conteneur avec sa propre IP sur le réseau Docker. Avec `kubectl get pods -A` j'ai vu que même sur un cluster vide, des pods système tournent déjà dans le namespace `kube-system`, comme coredns pour le DNS interne, traefik pour l'Ingress, local-path-provisioner pour le stockage et metrics-server pour les métriques.

## Exercice 2 — Premier déploiement

3. Quels objets Kubernetes ont été créés par `kubectl create deployment` ? (indice : `kubectl get all`)
   > La commande `kubectl create deployment` a créé trois objets imbriqués. Elle crée d'abord le Deployment `web`, qui crée à son tour un ReplicaSet `web-68d995574f`, et ce ReplicaSet crée le pod `web-68d995574f-hk72l`. Le nom du pod reprend d'ailleurs celui du ReplicaSet suivi d'un suffixe aléatoire. Le `service/web` visible dans `kubectl get all` ne vient pas de cette commande mais du `kubectl expose` lancé juste après, et le `service/kubernetes` existait déjà avant puisque c'est le service interne qui donne accès à l'API du cluster.
4. Vous avez supprimé le pod `web-...` : que s'est-il passé, et pourquoi ?
   >Dans le second terminal, j'ai vu mon pod `web-68d995574f-hk72l` passer en Terminating puis en Completed, et presque en même temps un nouveau pod `web-68d995574f-qqrmd` est apparu, passé par Pending et ContainerCreating avant d'être Running en 2 secondes. Le nom a changé mais seulement le suffixe, le début `web-68d995574f` est resté le même car c'est le même ReplicaSet qui l'a recréé. Je n'avais pas créé un simple pod mais un Deployment, et son ReplicaSet doit maintenir en permanence 1 pod en vie. Dès qu'il a constaté que l'état réel (0 pod) ne correspondait plus à l'état désiré (1 pod), il en a recréé un. C'est la boucle de réconciliation de Kubernetes, qui lui permet de se réparer tout seul.

## Exercice 3 — Modifier un conteneur en direct

5. Après suppression du pod, votre message « Bienvenue... » a-t-il survécu ? Pourquoi ?
   >Non, mon message n'a pas survécu. Après la suppression du pod et la relance du tunnel, j'ai retrouvé la page « Welcome to nginx! » d'origine. Le système de fichiers d'un conteneur est éphémère, ma modification n'existait que dans ce conteneur précis. Le ReplicaSet a recréé un pod neuf à partir de l'image `nginx` d'origine, qui ne contient pas mon changement. Modifier un conteneur à la main revient donc à perdre son travail au prochain redémarrage, il faut décrire le contenu voulu en dehors du conteneur.
6. Pourquoi le `port-forward` a-t-il cessé de fonctionner après la suppression du pod ?
   >Un `port-forward` vers un Service ne passe pas vraiment par le Service, il choisit un pod au démarrage et s'attache directement à lui. Quand ce pod a été supprimé, le tunnel a perdu sa cible. Dans `/tmp/web.log` j'ai vu l'erreur `failed to find sandbox ... not found` puis `error: lost connection to pod`, et le processus s'est arrêté avec Exit 1. Il ne bascule pas tout seul vers le nouveau pod, j'ai dû relancer le tunnel à la main.

## Exercice 4 — Déclaratif

7. Quelle différence entre `kubectl create` et `kubectl apply` ?
   >`kubectl create` est impératif. Je donne un ordre « crée cet objet », et si l'objet existe déjà la commande échoue avec une erreur AlreadyExists. `kubectl apply` est déclaratif. Je décris l'état voulu dans un fichier YAML, et Kubernetes compare ce fichier avec ce qui tourne pour créer l'objet s'il n'existe pas ou ne modifier que ce qui a changé. Je l'ai vu en modifiant ma page : au second `apply`, seule la ConfigMap est passée en `configured`, alors que le Deployment et le Service sont restés `unchanged`. On peut donc relancer `apply` autant de fois qu'on veut, ce qui permet de versionner les fichiers dans Git et de les rejouer, comme en production.
8. À quoi sert le `selector` d'un Service ? Que se passe-t-il s'il ne correspond à aucun pod ?
   >Le `selector` d'un Service indique à quels pods envoyer le trafic. Il cible tous les pods qui portent les labels indiqués, ici `app: site`, et Kubernetes tient à jour la liste de leurs IP. Chez moi, `kubectl get endpoints site` affiche `10.42.0.9:80`, l'IP de mon pod. J'ai eu un avertissement indiquant que l'objet Endpoints est déprécié depuis la v1.33 et remplacé par les EndpointSlices, qu'on consulte avec `kubectl get endpointslices -l kubernetes.io/service-name=site`. Si le selector ne correspond à aucun pod, par exemple à cause d'une faute de frappe dans le label, le Service est créé sans erreur mais n'a aucun endpoint. Le trafic n'arrive alors nulle part et le site est injoignable, c'est une panne silencieuse qu'on repère justement en regardant les endpoints.

## Exercice 5 — Scaling & mises à jour

9. Sur quels nœuds se trouvent vos 3 réplicas ? Collez la sortie de `kubectl get pods -l app=site -o wide`.
   ```
   NAME                    READY   STATUS    RESTARTS   AGE     IP          NODE
   site-5774d58f5c-97lzn   1/1     Running   0          27s     10.42.2.5   k3d-lab-agent-0
   site-5774d58f5c-cskcv   1/1     Running   0          27s     10.42.1.6   k3d-lab-server-0
   site-5774d58f5c-x8hmh   1/1     Running   0          6m58s   10.42.0.9   k3d-lab-agent-1
   ```
   > Kubernetes a réparti mes 3 réplicas sur les 3 nœuds, un par nœud, pour que la panne d'un nœud n'emporte pas tout le site. On voit qu'un pod est placé sur `k3d-lab-server-0`, ce qui confirme que dans k3s le control-plane accepte aussi des pods applicatifs. Le pod `x8hmh` a 6 minutes car il existait déjà, seuls les 2 manquants ont été créés.
10. Avec l'image `nginx:9.99`, quel statut avait le pod ? Quelle commande vous a donné la cause ?
    >Le nouveau pod `site-c759f856b-wth5d` est resté en 0/1. Il est d'abord passé en `ErrImagePull` puis en `ImagePullBackOff`. C'est la commande `kubectl describe pod site-c759f856b-wth5d` qui m'a donné la cause exacte dans la section Events, avec le message `failed to resolve reference "docker.io/library/nginx:9.99": not found`. Le tag 9.99 n'existe tout simplement pas sur Docker Hub. Le kubelet a fait plusieurs tentatives de téléchargement, puis il espace de plus en plus ses essais, d'où le BackOff.
11. Pendant la panne, le site restait-il accessible ? Pourquoi ?
    >Oui, le site est resté accessible. Le rolling update n'a créé qu'un seul pod en 9.99 et attendait qu'il soit prêt avant d'arrêter un ancien pod. Comme il n'a jamais démarré, mes 3 anciens pods sont restés en Running et le Service a continué à leur envoyer le trafic. Après le `rollout undo`, j'ai vérifié avec `kubectl diff -f k8s/site/` que mon fichier YAML et le cluster étaient de nouveau alignés, car c'est le dépôt Git qui doit faire foi.

## Exercice 6 — Portainer & Mario

12. URL publique de votre Mario :
    >https://jubilant-potato-4xqx46r77g5f759w-8585.app.github.dev/
13. Retrouvez le Deployment `mario` en ligne de commande. Portainer et `kubectl` parlent-ils au même composant ? Lequel ?
    >Avec `kubectl get deploy mario -o yaml`, je retrouve un Deployment Kubernetes tout à fait standard, avec la même structure que ceux créés en ligne de commande (selector, replicas, stratégie RollingUpdate). La seule différence est que Portainer a ajouté ses propres labels `io.portainer.kubernetes.application.*`, par exemple `owner: admin`, pour savoir qu'il l'a créé. C'est aussi pour ça que Portainer affiche `site` et `web` avec le badge External, puisque je les ai créés moi-même avec kubectl. Portainer et kubectl parlent donc au même composant, l'API Server du cluster. Portainer l'appelle depuis l'intérieur du cluster via l'adresse `https://kubernetes.default.svc` affichée dans son tableau de bord, avec le ServiceAccount `portainer-sa-clusteradmin` créé à l'installation, et kubectl l'appelle depuis mon Codespace. Ce n'est qu'une interface graphique de plus devant la même API.

## Bonus (facultatif)

14. Bonus réalisés et remarques :
    > J'ai réalisé les 4 bonus.
    >
    > **A, Ingress** : j'ai relié l'Ingress au Service `site` sur son port 80, et mon site répond sur le port 8081 via Traefik sans aucun port-forward. En supprimant les 3 pods d'un coup, j'ai eu deux réponses HTTP 502 pendant environ 2 secondes, puis le site est revenu tout seul en HTTP 200, alors qu'avec port-forward il fallait relancer le tunnel à la main.
    >
    > **B, Probes et ressources** : j'ai ajouté une readinessProbe HTTP sur `/` et des requests/limits (50m/100m CPU, 32Mi/64Mi mémoire). En faisant pointer la probe sur `/nope`, le nouveau pod restait Running mais 0/1 avec l'événement `Readiness probe failed: HTTP probe failed with statuscode: 404`. Il ne recevait aucun trafic et bloquait le rolling update, donc les anciens pods continuaient de servir le site. J'ai ensuite réappliqué mon fichier et vérifié avec `kubectl diff` que tout était de nouveau aligné.
    >
    > **C, Drain** : `k3d-lab-agent-0` est passé en SchedulingDisabled, ses pods (site, traefik, metrics-server) ont été évincés et recréés sur les autres nœuds, seul le DaemonSet svclb-traefik est resté. Après le `uncordon`, aucun pod n'est revenu sur ce nœud, car Kubernetes ne rééquilibre pas les pods existants, le scheduler ne place que les nouveaux.
    >
    > **D, Namespaces** : j'ai déployé une seconde copie du site dans un namespace `prod`. Avec `kubectl get deploy -A`, je vois deux Deployments qui s'appellent tous les deux `site`, l'un dans `default` et l'autre dans `prod`, chacun en 3/3 et sans conflit, car un nom n'a besoin d'être unique qu'à l'intérieur de son namespace.

---

**Score obtenu avec `./scripts/check.sh` :** 18 / 18
