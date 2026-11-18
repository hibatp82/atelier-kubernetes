# Rendu — Atelier Kubernetes

> Complétez ce fichier au fil de l'atelier, puis faites un commit et un push (voir la section « Rendu » du README).

**Nom / Prénom :** AQZIZ Hiba
**Groupe :** RS2

---

## Exercice 1 — Cluster

1. Combien de nœuds affiche `kubectl get nodes` ? Quel est le rôle de chacun ?
   >  3 nœuds tous en eeady :
   > k3d-lab-server-0 (c'est le control-plane). Il héberge l'API Server, le scheduler, le controller-manager et la base d'état du cluster. Il décide où placer les pods.
   > k3d-lab-agent-0 et k3d-lab-agent-1 (les workers). Ils exécutent les pods des app grâce au kubelet et au runtime de conteneurs containerd

2. Que montre `docker ps` ? Qu'est-ce qu'un « nœud » dans k3d, concrètement ?
 > docker ps montre 4 conteneurs :
      > k3d-lab-server-0, k3d-lab-agent-0, k3d-lab-agent-1 : image `rancher/k3s:v1.35.5-k3s1`
    >   k3d-lab-serverlb : image k3d-proxy, le load balancer qui relie le port 8081 du Codespace au port 80 du cluster et expose l'API Server
   Dans k3d un nœud est un conteneur docker qui exécute k3s. Les IP des nœuds sont celles des conteneurs sur le réseau Docker k3d-lab. On simule plusieurs machines sur une seule vm
> Sur un cluster vide les pods système sont dans le namespace kube-system


## Exercice 2 — Premier déploiement

3. Quels objets Kubernetes ont été créés par `kubectl create deployment` ? (indice : `kubectl get all`)
   >ça a a créé 3 objets :
   > un deployment `deployment.apps/web` : il décrit l'état voulu
   > un replicaSet `replicaset.apps/web-68d995574f` : créé par le Deployment, il maintient le bon nombre de pods
   > un pod `pod/web-68d995574f-zcjqk` : créé par le ReplicaSet, c'est lui qui exécute le conteneur nginx
   > le nom du pod se lit ainsi : nom du Deployment et hash du ReplicaSet puis suffixe aléatoire.
   > Le Service `service/web` vient de la commande kubectl expose Le Service `kubernetes` existe par défaut, il expose l'API Server.
4. Vous avez supprimé le pod `web-...` : que s'est-il passé, et pourquoi ?
> Le pod `web-68d995574f-zcjqk` est passé en `Terminating` puis `Completed`. Un nouveau pod web-68d995574f-djlf6 a été créé (`Pending`, puis `ContainerCreating`, puis `Running` en 2 secondes). Le nom a changé, mais le hash du ReplicaSet `68d995574f` est resté le même
car on n'a pas créé un pod isolé mais un Deployment. Son ReplicaSet doit maintenir 1 réplica. Quand il voit 0 pod au lieu de 1, il en recrée un

## Exercice 3 — Modifier un conteneur en direct

5. Après suppression du pod, votre message « Bienvenue... » a-t-il survécu ? Pourquoi ?
   > Non après la relance du tunnel la page affiche de nouveau bienvenue
   > Le système de fichiers d'un conteneur est éphémère. La modification faite avec `kubectl exec` n'existait que dans le conteneur du pod supprimé. Le ReplicaSet a recréé un pod neuf à partir de l'image `nginx` d'origine qui ne contient pas ma modification.
   > Il ne faut donc jamais modifier un conteneur à la main , le contenu doit être décrit en dehors du conteneur, dans l'image.

6. Pourquoi le `port-forward` a-t-il cessé de fonctionner après la suppression du pod ?
   > Même lancé sur svc/web un kubectl port-forward ne passe pas vraiment par le service, il choisit un seul pod derrière le Service au démarrage et s'y attache directement
   > Quand ce pod a été supprimé, son réseau a été détruit. Le log `/tmp/web.log` nos montre que network namespace for sandbox ... is clos puis `error: lost connection to pod`, et le tunnel s'est arrêté (`Exit 1`).
   > Le tunnel ne bascule pas tout seul vers le nouveau pod il faut le relancer

## Exercice 4 — Déclaratif

7. Quelle différence entre `kubectl create` et `kubectl apply` ?
   >
8. À quoi sert le `selector` d'un Service ? Que se passe-t-il s'il ne correspond à aucun pod ?
   >

## Exercice 5 — Scaling & mises à jour

9. Sur quels nœuds se trouvent vos 3 réplicas ? Collez la sortie de `kubectl get pods -l app=site -o wide`.

   NAME                   READY   STATUS    RESTARTS   AGE     IP          NODE               NOMINATED NODE   READINESS GATES
   site-75775484b-dwzg6   1/1     Running   0          22s     10.42.1.6   k3d-lab-agent-1    <none>           <none>
   site-75775484b-q6kwh   1/1     Running   0          3m22s   10.42.0.6   k3d-lab-agent-0    <none>           <none>
   site-75775484b-vt6fj   1/1     Running   0          22s     10.42.2.7   k3d-lab-server-0   <none>           <none>

   > Le scheduler a réparti les 3 réplicas sur 3 nœuds différents : agent-0, agent-1 et server-0. Il étale les pods pour qu'une seule panne de nœud ne coupe pas le service. Avec k3s, le control-plane accepte aussi des pods, ce qui n'est pas le cas par défaut sur un cluster Kubernetes classique.
   > Pendant le rolling update vers nginx:1.28, j'ai observé dans `kubectl get pods -o wide -w` que chaque nouveau pod (ReplicaSet `site-5cc664db9c`) passait en Running avant qu'un ancien pod (ReplicaSet `site-75775484b`) passe en Terminating. Les pods ont été remplacés un par un, sans coupure.
10. Avec l'image `nginx:9.99`, quel statut avait le pod ? Quelle commande vous a donné la cause ?
   > Le nouveau pod `site-fc576997-vb9cq` (un nouveau ReplicaSet `site-fc576997`) est resté en `0/1 ErrImagePull`, puis `ImagePullBackOff`.
    > La commande `kubectl describe pod site-fc576997-vb9cq` donne la cause dans la section Events : `Failed to pull image "nginx:9.99" ... not found`. Ce tag n'existe pas sur Docker Hub.
11. Pendant la panne, le site restait-il accessible ? Pourquoi ?
     > Oui , les 3 anciens pods en nginx:1.28 (`site-5cc664db9c-cwlf4`, `-hhqwq`, `-nmvrp`) sont restés `1/1 Running`.
    > Un rolling update ne supprime un ancien pod qu'une fois le nouveau prêt. Le pod 9.99 n'ayant jamais démarré, aucun ancien pod n'a été retiré, et le Service a continué à envoyer le trafic aux pods sains. Avec la stratégie par défaut (maxUnavailable 25 %, soit 0 pod sur 3), Kubernetes ne descend jamais sous 3 pods disponibles.
    > `kubectl rollout undo` est ensuite revenu à nginx:1.28. `kubectl diff -f k8s/site/` n'affichait plus rien : le cluster correspondait de nouveau aux fichiers YAML.

## Exercice 6 — Portainer & Mario

12. URL publique de votre Mario :
    >
13. Retrouvez le Deployment `mario` en ligne de commande. Portainer et `kubectl` parlent-ils au même composant ? Lequel ?
    >

## Bonus (facultatif)

14. Bonus réalisés et remarques :
    >

---

**Score obtenu avec `./scripts/check.sh` :** ... / ...
