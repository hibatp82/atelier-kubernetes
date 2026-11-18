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
   ```

   ```
10. Avec l'image `nginx:9.99`, quel statut avait le pod ? Quelle commande vous a donné la cause ?
    >
11. Pendant la panne, le site restait-il accessible ? Pourquoi ?
    >

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
