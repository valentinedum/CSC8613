# Rapport TP1

## Exercice 1 : Installation de Docker et vérification de l'environnement

Docker a été installé comme le montre l'éxéxution de la commande suivante:
```bash
docker run hello-world
```

![docker_hello](./images_tp1/Capture%20d’écran%202025-12-05%20104537.png)

On peut vérifier que le conteneur s'est exécuté correctement:
```bash
docker ps -a
```

![docker_exec](./images_tp1/Capture%20d’écran%202025-12-05%20104655.png)

## Exercice 2 : Premiers pas avec Docker : images et conteneurs

Une image Docker est un modèle pour faire des conteneurs Docker. Elle est statique et contient tout ce qu'il faut pour que l'application fonctionne. Le conteneur est l'instance en exécution de cette image. On obtient une image en exécutant un DockerFile qui contient les instructions de création. Puis on obtient le conteneur en runnant notre image.

![docker_principle](./images_tp1/unnamed.png)

Nous allons exécuté la commande suivant:
```bash
docker run alpine echo "Bonjour depuis un conteneur Alpine"
```

Voici ce qu'on a après exécution:
![docker_alpine](./images_tp1/image.png)

Docker n'a pas réussi à trouver l'image en local, donc il la cherche dans ses librairies Docker Hub, la télécharge. Tout s'est bien exécuté puisqu'on a eu le echo. Enfin le conteneur s'est arrété.

Maintenant, on peut remarquer que 2 conteneurs se sont lancés et arrétés :
```bash
docker ps -a
```

![docker_stop](./images_tp1/Capture%20d’écran%202025-12-05%20112315.png)

Nos conteneurs se sont arrêtés car ils ne vivent que tant que leur processus principal est en cours d'exécution. Leurs images ayant été téléchargées et leurs commandes ayant été réalisées, les conteneurs se terminent. Pourtant, ils ne sont pas supprimés, ils sont juste à l'arrêt.

A présent, nous lançons le conteneur en mode interractif
et nous tapons des commandes à l'intérieur.
![docker_interractif](./images_tp1/Capture%20d’écran%202025-12-05%20112836.png)

On observe que l'invité de commande a changé. On est sur la root du conteneur et on voit ce qui s'y trouve avec ```ls```. Donc le conteneur est bien isolé de ma machine.

Avec ```uname -a```, on comprend que le conteneur partage portant bien le même OS que ma machine hôte **WSL2**.

## Exercice 3: Construire une première image Docker avec une mini-API FastAPI

On construite une API dans le fichier /api/app.py. Pour cela, il est nécessaire de configurer la Dockefile pour créer notre image Docker avec toutes les dépendances nécessaires à notre application.

Une fois les 2 créés, on va lancer le build du docker. Pour cela, on se place dans le dossier api et on lance:
```bash
docker build -t simple-api .
```

![docker_api](./images_tp1/Capture%20d’écran%202025-12-05%20114840.png)

Le build s'est bien terminé.

## Exercice 4: Exécuter l’API FastAPI dans un conteneur Docker

On lance maintenant un conteneur en exécutant l'image simple api:
```bash
docker run -p 8000:8000 simple-api
```

Grâce à cette commande docker a runné le conteneur sur son port 8000 mais la exposé en local sur le port 8000 de ma machine. On peut donc y accéder depuis un moteur de recherche.
Tout à l'air de fonctionné -> code 200

![docker_test](./images_tp1/Capture%20d’écran%202025-12-05%20115856.png)

On peut tester que le endpoint **/health** nous renvoie bien ce qu'il faut.
![api_test](./images_tp1/Capture%20d’écran%202025-12-05%20115909.png)

La réponse JSON est bien celle attendue.


Maintenant, nous allons afficher la liste des conteneurs en cours d'execution.

![docker_conteneurs](<images_tp1/Capture d’écran 2025-12-05 120210.png>)

Il n'y a que le conteneur simple-api en cours d'exécution. Avec comme nom **upbeat-galileo**, comme image utilisée **simple-api** et comme port mappé **0.0.0.0:8000->8000/tcp**.

On stoppe maintenant le conteneur en récupérant son id **28e877056d1e**.

![docker_kill](./images_tp1/Capture%20d’écran%202025-12-05%20120248.png)

En effet, il n'apparait plus dans ```docker ps```. On peut, cependant voir qu'il est encore disponible dans ```docker ps -a```

![docker_conteneurs_a](./images_tp1/Capture%20d’écran%202025-12-05%20134307.png)

En réalité,  ```docker ps``` affiche les conteneurs en cours d'exécution et ```docker ps -a``` liste les conetneurs qu'ils soient actifs, arrêtés ou en erreur. Ces conteneurs existent toujours sur le disque dur.

## Exercice 5: Démarrer un mini-système multi-conteneurs avec Docker Compose

On va créer un fichier docker-compose.yml à la racine pour permettre de lancer plusieurs services en une commande.

Puis nous le remplissons afin de définir un service db basé sur l'image officielle de postgres:16, un service api qui utilise notre Dockerfile, des variables d'environnements, les ports nécessaires. 

Nous lançons tous les services en arrière plan.
![docker_compose](./images_tp1/Capture%20d’écran%202025-12-05%20135247.png)

Les deux services db et api sont bien en train de tournés.
![docker_services](./images_tp1/Capture%20d’écran%202025-12-05%20135425.png)

On peut vérifier que l'API est bien lancée:
![docker_api](./images_tp1/Capture%20d’écran%202025-12-05%20135425.png)

Puis on arrête les conteneurs avec:
```bash
docker compose down
```

![docker_down](./images_tp1/Capture%20d’écran%202025-12-05%20140101.png)

Cette commande est différente de ```docker stop``` qui se contente d'arrêter l'exécution d'un conteneur spécifique tout en le gardant sur le disque. ```docker compose down```  arrête tous les services de l'application et supprime définitivement les conteneurs associés. Mais il ne détruit pas les volumes (données présentes dans la db par exemple) comme ```docker compose down -v```

## Exercice 6: Interagir avec la base de données PostgreSQL dans un conteneur

On va redémarrer le Docker compose.

![docker_recompose](./images_tp1/Capture%20d’écran%202025-12-05%20140726.png)

Puis on va ouvrir un shell psql à l'intérieur du conteneur PostgreSQL avec la commande
```bash
docker compose exec db psql -U demo -d demo
```

```exec``` permet de lancer une commande à l'intérieur du conteneur déjà en cours ```db```. ```-U demo``` est une option de la commande psql, où l'on spécifie l'utilisateur qui se connecte à la base PostgreSQL et ```-d demo``` est aussi une option qui spécifie le nom de la base de donnée à laquelle se connecter.

Ensuite, on exécute les requêtes ci-dessous:
![docker_psql](./images_tp1/Capture%20d’écran%202025-12-05%20141118.png)

Donc la première requête cherche à afficher la version de PostgreSQL. Dans notre cas c'est celle spécifiée dans le docker-compose.yml -> 16.
La deuxième requête confirme qu'on s'est connecté à la base demo.

Depuis un autre service Docker soit l'api, on peut se connecter à la base avec les paramètres suivants:
- hostname : **db**
- port: **5432**
- utilisateur: **demo**
- mot de passe: **demo**
- bdd: **demo**
Ce sont les valeurs spécifiées dans le docker-compose.yml.