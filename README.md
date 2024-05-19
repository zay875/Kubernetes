#Coder une application web composée de deux services web :un frontend et un backend et faire communiquer les deux.#

#Deux services:

●	Front-end.
●	Back-end.
Le code source est disponible sur Github:
https://github.com/charroux/CodingWithKubernetes

1.	Définition de l'application
Back-end
L'application back-end attend une requête HTTP GET sur /hello et renvoie la réponse World!
Front-end
Pour le front-end, nous allons utiliser l'image Docker existante située dans le répertoire spécifié https://hub.docker.com/u/efrei, “efrei/front-end”.
L'application front-end est un service Spring Boot qui fonctionne en tant que frontend et communique avec un backend via une requête HTTP GET sur /. Elle affiche "Hello" concaténé avec le résultat de l'application back-end en injectant une dépendance : @Value("${backEndURL}").
2.	Déploiement et Création des Services Back-end et Front-end avec Kubernetes
Nous allons créer un seul fichier YAML, front-back-app.yml, pour le déploiement et la création des services.
Déploiement et Service Front-end
●	Déploiement : Utilise l'image disponible sur Docker Hub à l'adresse https://hub.docker.com/u/efrei, image : efrei/front-end:1.
●	Service : Expose ce service via type: NodePort et targetPort: 8080.
Déploiement et Service Back-end
●	Déploiement : Utilise l'image que nous avons créée, zin0096/back-end.
●	Service : Puisque ce service est accédé via le service front-end, il est exposé via type: ClusterIP et targetPort: 8080.

 
3.	Comment le frontend peut-il accéder au backend ?
●	Utilisation du DNS dans Kubernetes
Kubernetes propose un service DNS cluster addon qui attribue automatiquement des noms DNS aux autres services. Dès que le service DNS fonctionne, l'application front-end peut demander au DNS de résoudre l'adresse back-end.
•	Principe des Douze Facteurs
Stocker les adresses de service en tant que constantes dans le code est une violation du principe des douze facteurs. Selon ce principe, les configurations, y compris les adresses de service, doivent être externalisées et injectées dans l'application.
●	Configuration de l'Adresse du Back-end
Dans cette configuration, l'adresse du back-end est stockée dans un fichier de propriétés du frontend. Vous pouvez voir l'exemple sur GitHub : application.yml.
●	Injection de la Propriété
L'adresse du back-end est injectée dans l'application frontend avec :
@Value("${backEndURL}") avec 
backEndURL: http://back-end-service.default.svc.cluster.local:80/hello
Verification de la configuration du DNS

 
1.1.	Extraire l’adresse IP externe
>Minikube service front-end-service –url 
 
1.2.	Tester l’application
127.0.0.1/65475/
 
On voit bien le résultat affiche” hello(from the front-end) “concaténé avec “Word!(from the back-end)”.


Servicemesh-kubernetes
1.	Installation de Istio:

Istio est un maillage de services (Service Mesh) open source utilisé pour la connexion, la sécurité, l'observabilité et la gestion du trafic entre les microservices. Il injecte automatiquement des proxies Sidecar (comme Envoy) pour intercepter et gérer le trafic entre les services. 
Le rôle d'Istio dans Kubernetes comprend principalement : la simplification de la gestion du trafic des microservices (comme l'équilibrage de charge et le routage), le renforcement de la sécurité entre les services (comme le chiffrement mTLS), la fourniture d'observabilité (comme les journaux et la surveillance) et la mise en œuvre du contrôle des politiques (comme la limitation de débit et l'authentification). 
Ces fonctionnalités aident les développeurs et les équipes d'exploitation à mieux gérer et optimiser l'architecture des microservices.
Dans ce projet, nous utilisons Istio principalement parce qu'il prend en charge l'API Kubernetes Gateway et peut être utilisé comme API par défaut pour la gestion future du trafic.


Tout d’abord, on utilise les commandes ci-dessous pour télécharger Istio.
curl -L https://istio.io/downloadIstio | sh -
cd istio-<version>
export PATH=$PWD/bin:$PATH

Ensuite, on installe le Istio dans votre cluster Kubernetes par le command istioctl install --set profile=demo -y

Grâce à la commande “kubectl label namespace default istio-injection=enabled”, les pods dans l'espace de noms par défaut acquièrent automatiquement les fonctionnalités du maillage de services Istio, leur permettant de participer au maillage de services Istio. 

Pour vérifier si le déploiement de istio est bien mit dans le Kubernetes, on exécute la commande “kubectl -n istio system get deployments”, comme la photo montre ci-dessous, les deployment de istio sont bien mis.
 

Ensuite, on crée un fichier infrastructure.yaml pour notre application, il est composé par 4 parties: les codes deployments de front-end et de back-end, les codes de services de les deux service, les codes de services virtuels de les deux services et le code de gateway avec le selecteur : istio.ingressgateway.
Voici quelques parties de le fichier infrastructure.yaml
  
  

Pour la partie de Gateway, cette configuration définit une passerelle Istio nommée microservice-gateway avec les fonctionnalités suivantes : tout d'abord, elle sélectionne l'instance de passerelle, s'appliquant aux instances de passerelle Istio portant l'étiquette istio=ingressgateway. Ensuite, en ce qui concerne la configuration du port, cette passerelle écoute le trafic HTTP sur le port 8080. Enfin, pour la configuration des hôtes, cette passerelle accepte les requêtes provenant de n'importe quel hôte (hosts: ["*"]). Grâce à cette configuration de passerelle, on peut contrôler quels flux de trafic peuvent entrer dans le maillage de services Istio et les traiter via l'instance istio-ingressgateway.

Pour la partie de service virtual, cette configuration définit deux Istio VirtualService nommé front-app et back-app avec les fonctionnalités suivantes : concernant la configuration de l'hôte, il s'applique à tous les hôtes (hosts: ["*"]). Pour la configuration de la passerelle, il traite le trafic via l'Istio Gateway nommé microservice-gateway. Pour les règles de routage HTTP, d'abord, il correspond à toutes les requêtes commençant respectivement par /front et /back; ensuite, il réécrit l'URI respectivement correspondant de /front et /back à / ; enfin, il achemine les requêtes vers les services front-end-service.default.svc.cluster.local et back-end-service.default.svc.cluster.local sur le port 80. Grâce à cette configuration de VirtualService, il est possible de contrôler le trafic entrant dans microservice-gateway, de correspondre et de réécrire des URI spécifiques, puis de router le trafic vers le service backend désigné.

De plus, on applique le fichier de déploiement par la commande “kubectl apply -f infrastructure.yaml”
 


Obtenir l’adresse du gateway:
Ensuite, rediriger le port vers la passerelle d'entrée d'Istio :
 

Après avoir confirmé que toutes les configurations sont correctes, nous accédons à l'URL suivante via le navigateur : http://localhost:31380/front/back.
Chaque fois quand on accède àl’URL pour demander connecter à 31380, il va afficher dans notre terminal.
 

 
Comme vous pouvez le voir dans l'image ci-dessus, notre lien ne fonctionne pas correctement et ce message d'erreur indique que la demande est en cours de réinitialisation avant d'atteindre le service backend, probablement en raison d'un problème de connexion ou d'un dysfonctionnement du service backend.
Nous avons essayé de résoudre le problème. Tout d'abord, nous avons vérifié l'état des pods et des services pour nous assurer que les pods et les services front-end et back-end fonctionnaient.
 

Ensuite, nous avons vérifié si la ressource Ingress était configurée correctement et le résultat était qu'elle était correctement configurée.
Nous avons également vérifié tous les fichiers que nous avons utilisés pour détecter les erreurs, mais malheureusement, aucune solution n'a finalement été trouvée.

 




























Déploiement d'un conteneur MySQL dans un cluster Kubernetes.
1.	Deploiement:
Les fichiers Yaml:
>kubectl apply -f mysql-secret.yaml; le fichier mysql-secret.yaml continent le mot de passe de connexion à la base de donnée.
>kubectl apply -f mysql-storage.yaml; ce fichier contient des informations sur le volume de stockage alloué.
>kubectl apply -f mysql-deployment.yaml; ce fichier crée un déploiement sur le cluster Kubernetes en utilisant une image de MySql mysql:5.7.
2.	Service:
Nous allons créer un service type NodePort, pour pouvoir accéder au service depuis l'extérieur du cluster et il sera également accessible de l'intérieur du cluster.
>kubectl apply -f mysql-serviceNodePort.yaml
3.	Verification de la configuration:
>kubectl get secrets
 

>kubectl get PersistentevVolumes
 
>kubectl get deployments
 
>kubectl get services
 
4.	Connexion au server MySQL:
>kubectl exec --stdin --tty mysql-77f69cd7b4-sq6db -- mysql -ptest1234
5.	Création de la base de donnée:
mysql> create database db_example;
6.	Création d’un utilisateur et lui accorder les droits:
mysql> create user 'springuser'@'%'identified by 'Password';
mysql> grant all on db_example.* to 'springuser'@'%';
Query OK, 0 rows affected (0.00 sec)
 
7.	Accéder au service de base de donnée:
>minikube service mysql --url
 
La tentative de connexion a échoué, ce qui n'était pas attendu,
On a essayé de faire le port Forwarding, mais le résultat n’a pas changé.

Une autre manière d'accéder au service est le port forwarding, pour rediriger le trafic du node Port vers un port local disponible

>kubectl port-forward mysql-77f69cd7b4-sq6db :31281

 

Comme on le voit, le service n'est toujours pas accessible de l'extérieur.
Néanmoins nous allons tester l’accessibilité à l'intérieur du cluster.
8.	Tester la connectivité:
Lister les pods pour obtenir le nom et l'IP :
 

Accéder au shell du pod front-end  et faire un ping au service mysql :
/ # ping 10.244.0.249
 

Le ping a bien réussi.
Conclusion :
En explorant les diverses fonctionnalités de Kubernetes, telles que les déploiements, les services, les configurations de réseau et les diagnostics, nous avons pu acquérir une compréhension approfondie de la manière dont les conteneurs sont gérés et orchestrés à grande échelle. L'application de ces connaissances dans un environnement réel nous a offert une expérience pratique précieuse, renforçant notre compétence dans l'utilisation de Kubernetes et Docker.
