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