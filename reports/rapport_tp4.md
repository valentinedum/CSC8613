# Rapport TP4

## Exercice 1 :  Mise en route + rappel de contexte (sanity checks + où on en est dans la pipeline)

Pour commencer ce TP, nous réinstallons la stack Docker Compose et vérifions que tout est bien lancé :

![docker_compose](./images_tp4/Capture%20d’écran%202025-12-17%20103450.png)

L'objectif de ce TP étant d'intégrer dans notre pipeline, l'entrainement du modèle, le suivi de l'execution du versionning et la mise en production de ce modèle, nous ajoutons un service MLflow dans notre `docker-compose.yml`. Puis nous redémarrons la stack.

```bash
 ✔ Container csc8613-postgres-1  Running                                                                                                                                          0.0s 
 ✔ Container csc8613-mlflow-1    Started                                                                                                                                          6.3s 
 ✔ Container csc8613-feast-1     Started                                                                                                                                          6.1s 
 ✔ Container csc8613-prefect-1   Started                                                                                                                                          6.3s 
 ✔ Container csc8613-api-1       Started 
```

MLflow est bien lancé ainsi que postgres, feast, prefect et l'API.
Tous sont lancés car sont décrit dans les services du Docker compose.

Nous pouvons vérifier que nous accédons bien aux deux endpoints (API et MLflow).

![mlflow](./images_tp4/Capture%20d’écran%202025-12-17%20104633.png)

![api](./images_tp4/Capture%20d’écran%202025-12-17%20104725.png)

Nous checkons aussi que nous accédons toujours à l'endpoint `features/{user_id}`.

![api_user](./images_tp4/Capture%20d’écran%202025-12-17%20105126.png)


## Exercice 2 : Créer un script d’entraînement + tracking MLflow (baseline RandomForest)

### Création du script d'entrainement et exécution
Dans cet exercice, nous allons compléter un script train_baseline.py
qui :
1. construit un dataset d’entraînement via des features déjà disponibles,
2. entraîne un modèle baseline,
3. trace l’exécution dans MLfow (params, métriques, artefacts),
4. enregistre le modèle dans le Model Registry.

Lorsque nous enregistrons le modèle dans MLflow, il faut bien penser à enregistrer toute la pipeline (preprocessing + modèle). Dans notre code on enregistre donc `pipe` et non `clf`.
En production, nous recevons les données brutes du FeatureStore, si on enregistrait seulement `clf`, les données brutes passeraient dans le modèle qui attends des données transformées.

Pour checker notre script, nous exécutons le conteneur prefect sur month_000 (donc avec `TRAIN_AS_OF=2024-01-31`) avec :

```bash
docker compose exec -e TRAIN_AS_OF=2024-01-31 prefect python /opt/prefect/flows/train_baseline.py
```

### Interprétation de l'exécution

Après quelques warnings nous obtenons dans le terminal :

```bash
[OK] Trained baseline RF. AUC=0.6208 F1=0.0524 ACC=0.7535 (run_id=81d1fbefdbd4465c81f085bdf80f6d00)
2025/12/17 11:19:32 INFO mlflow.tracking._tracking_service.client: 🏃 View run rf_baseline_2024-01-31 at: http://mlflow:5000/#/experiments/1/runs/81d1fbefdbd4465c81f085bdf80f6d00.
2025/12/17 11:19:32 INFO mlflow.tracking._tracking_service.client: 🧪 View experiment at: http://mlflow:5000/#/experiments/1.
```
Notre exécution se termine bien en `[OK]` avec un `run_id=81d1fbefdbd4465c81f085bdf80f6d00`.
Dans cette log nous retrouvons aussi le score des 3 métriques `AUC=0.6208 F1=0.0524 ACC=0.7535`

C'est dans l'interface de MLflow que nous retrouvons nos colonnes catégorielles et numériques :

![feature_schema](./images_tp4/Capture%20d’écran%202025-12-17%20134656.png)

Une seule des features est numérique. Ce qui parrait cohérent car dans le dataset, toutes les autres features étaient numériques ou prenaient 2 valeurs (booléen, ou 2 strings différents) sauf `net_service` qui prend au moins 3 valeurs **Fiber Optic**, **DSL**, **No**.


Ansi que les 3 métriques calculées (AUC, F1, ACC) et le temps d'entrainement `train_time_sec=1.5825`

![metrics_time](./images_tp4/Capture%20d’écran%202025-12-17%20134719.png)

J'ai fait le choix de rajouter un `mlflow.log_param` pour logger le nombre de lignes dans le dataset `training_rows=7043`


### Importance de `AS_OF` et `random_state`

On fixe `AS_OF`, pour pouvoir récupérer les même donnée d'entrainement à chaque exécution du script. Sans ça, le script récupererait des données différentes (les plus récentes), ce qui rendrait impossible la reproducibilité.

Enfin `random_state` est fixé pour garantir la reproducibilité des opérations aléatoires. Ainsi d'une exécution à une autre, chaque opération aléatoire sera exactement la même qu'à l'exécution précente. Donc l'entrainement, le split etc seront les mêmes donc les métriques aussi.

Ces deux paramètres sont très important en MLOps.

## Exercice 3 : Explorer l’interface MLFlow et promouvoir un modèle

Nous cherchons sur l'interface MLflow, notre dernière expérience en passant par `streamflow`, ensuite le dernier run en date.



