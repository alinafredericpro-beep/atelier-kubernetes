# Rendu — Atelier Kubernetes

> Complétez ce fichier au fil de l'atelier, puis faites un commit et un push (voir la section « Rendu » du README).

**Nom / Prénom :**
**Groupe :**

---

## Exercice 1 — Cluster

1. Combien de nœuds affiche `kubectl get nodes` ? Quel est le rôle de chacun ?
   > k3d-lab-serverlb   ghcr.io/k3d-io/k3d-proxy:5.9.0   Up 2 minutes
k3d-lab-agent-1    rancher/k3s:v1.35.5-k3s1         Up 2 minutes
k3d-lab-agent-0    rancher/k3s:v1.35.5-k3s1         Up 2 minutes
k3d-lab-server-0   rancher/k3s:v1.35.5-k3s1         Up 2 minutes

3 nœuds : k3d-lab-server-0 est le control-plane (master). Il héberge l’API Server, prend les décisions et place les pods. k3d-lab-agent-0 et k3d-lab-agent-1 sont les workers : ils exécutent les pods des applications.

2. Que montre `docker ps` ? Qu'est-ce qu'un « nœud » dans k3d, concrètement ?
   >Docker ps montre un conteneur Docker par nœud (server-0, agent-0, agent-1), plus un conteneur k3d-lab-serverlb qui sert de load balancer. Dans k3d, un « nœud » est donc concrètement un conteneur Docker qui fait tourner k3s, une distribution Kubernetes légère.

## Exercice 2 — Premier déploiement

3. Quels objets Kubernetes ont été créés par `kubectl create deployment` ? (indice : `kubectl get all`)
   >pod/web-68d995574f-cg79b   1/1     Running   0          3m46s

NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.43.0.1       <none>        443/TCP   4m43s
service/web          ClusterIP   10.43.134.117   <none>        80/TCP    3m43s

NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/web   1/1     1            1           3m46s

NAME                             DESIRED   CURRENT   READY   AGE
replicaset.apps/web-68d995574f   1         1         1       3m47s

Un Deployment web, qui a créé un ReplicaSet web-xxxx, qui a lui-même créé un Pod web-xxxx-yyyy. La commande kubectl expose a ensuite ajouté un Service web de type ClusterIP.

4. Vous avez supprimé le pod `web-...` : que s'est-il passé, et pourquoi ?
   >Le pod a été supprimé (statut Terminating), puis un nouveau pod est apparu immédiatement, avec un nom différent. Le ReplicaSet doit maintenir 1 réplica : il a constaté l’écart entre l’état désiré (1 pod) et l’état réel (0 pod) et l’a corrigé. C’est la boucle de réconciliation de Kubernetes.
   
## Exercice 3 — Modifier un conteneur en direct

5. Après suppression du pod, votre message « Bienvenue... » a-t-il survécu ? Pourquoi ?
   >Non, le message a disparu et la page nginx par défaut est revenue. Le système de fichiers d’un conteneur est éphémère : le nouveau pod repart de l’image nginx d’origine. Les modifications faites à la main dans un conteneur sont perdues à chaque recréation.

6. Pourquoi le `port-forward` a-t-il cessé de fonctionner après la suppression du pod ?
   >Un port-forward vers un Service s’attache en réalité à un seul pod précis. Quand ce pod est supprimé, la connexion est perdue et le tunnel s’arrête : /tmp/web.log affiche une erreur. Il faut relancer le tunnel pour qu’il s’accroche au nouveau pod.


## Exercice 4 — Déclaratif

7. Quelle différence entre `kubectl create` et `kubectl apply` ?
   >kubectl create est impératif : on donne un ordre (« crée cet objet »). Si l’objet existe déjà, la commande échoue (AlreadyExists) et elle ne sait pas le mettre à jour. L’état du cluster n’est écrit nulle part : il résulte d’une suite de commandes difficile à reproduire.
   
   kubectl apply -f est déclaratif : on décrit l’état désiré dans un fichier YAML. Kubernetes compare ce fichier à l’état réel et crée l’objet s’il n’existe pas, ou ne modifie que ce qui a changé. On peut relancer la commande autant de fois qu’on veut, avec le même résultat (elle est idempotente). Les fichiers peuvent être versionnés dans Git, relus et rejoués à l’identique sur un autre cluster : c’est la méthode utilisée en production (GitOps).

8. À quoi sert le `selector` d'un Service ? Que se passe-t-il s'il ne correspond à aucun pod ?
   >Le selector indique au Service vers quels pods envoyer le trafic, grâce à leurs labels (ici app: site). Le Service tient à jour en permanence la liste des IP des pods Ready qui portent ce label : ce sont ses Endpoints. Comme les pods sont éphémères et changent d’IP, le selector permet au Service de toujours retrouver les bons pods, sans connaître leurs noms ni leurs adresses.

   S’il ne correspond à aucun pod (faute de frappe dans le label, par exemple), le Service est bien créé mais n’a aucun endpoint : kubectl get endpoints site affiche <none>. Le trafic n’est envoyé nulle part et le site est inaccessible, même si les pods sont en Running. Aucune erreur n’apparaît à la création, il faut donc vérifier les endpoints.

## Exercice 5 — Scaling & mises à jour

9. Sur quels nœuds se trouvent vos 3 réplicas ? Collez la sortie de `kubectl get pods -l app=site -o wide`.
   ```
Les 3 réplicas sont répartis sur plusieurs nœuds (agent-0, agent-1…). C’est le scheduler de Kubernetes qui a choisi leur emplacement, en étalant les pods pour la tolérance aux pannes.

`kubectl get pods -l app=site -o wide`.

NAME                   READY   STATUS    RESTARTS   AGE     IP          NODE               NOMINATED NODE   READINESS GATES
site-75775484b-ddfw4   1/1     Running   0          8m36s   10.42.2.6   k3d-lab-agent-0    <none>           <none>
site-75775484b-lp2mf   1/1     Running   0          110s    10.42.1.5   k3d-lab-server-0   <none>           <none>
site-75775484b-x6kb9   1/1     Running   0          110s    10.42.0.9   k3d-lab-agent-1    <none>           <none>

   ```
10. Avec l'image `nginx:9.99`, quel statut avait le pod ? Quelle commande vous a donné la cause ?
    >$ kubectl get pods -l app=site

NAME                   READY   STATUS         RESTARTS   AGE
site-75775484b-ddfw4   1/1     Running        0          11m
site-75775484b-lp2mf   1/1     Running        0          4m59s
site-75775484b-x6kb9   1/1     Running        0          4m59s
site-fc576997-gk6w7    0/1     ErrImagePull   0          4s

Pour avoir la cause : 
@alinafredericpro-beep ➜ /workspaces/atelier-kubernetes (main) $ kubectl describe pod site-fc576997-gk6w7  



11. Pendant la panne, le site restait-il accessible ? Pourquoi ?
    >Oui, le site restait accessible. Lors d’un rolling update, Kubernetes ne supprime les anciens pods que lorsque les nouveaux sont prêts. Le pod en nginx:9.99 n’étant jamais prêt, la mise à jour s’est bloquée et les anciens pods (nginx:1.28) ont continué à servir le site. De plus, le Service n’envoie le trafic qu’aux pods Ready, donc le pod cassé ne recevait aucune requête.


## Exercice 6 — Portainer & Mario

12. URL publique de votre Mario :
    >https://solid-space-engine-gxppxrjpvqwghp9qg-8585.app.github.dev/
    
13. Retrouvez le Deployment `mario` en ligne de commande. Portainer et `kubectl` parlent-ils au même composant ? Lequel ?
    >
   Portainer et kubectl parlent au même composant : l’API Server du control plane (nœud k3d-lab-server-0). kubectl l’appelle depuis le terminal grâce au fichier ~/.kube/config. Portainer, qui tourne dans le cluster, l’appelle avec son ServiceAccount, qui a les droits admin. L’API Server enregistre l’objet dans etcd, puis les contrôleurs, le scheduler et le kubelet font le reste, quel que soit l’outil utilisé. Portainer n’est qu’une interface graphique au-dessus de la même API : ce qui est créé dans l’un est immédiatement visible dans l’autre.



## Bonus (facultatif)

14. Bonus réalisés et remarques :
    >

---

**Score obtenu avec `./scripts/check.sh` :** ... / ...
