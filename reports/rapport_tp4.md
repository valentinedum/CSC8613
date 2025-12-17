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
Model name: streamflow_churn, version 3
Created version '3' of model 'streamflow_churn'.
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

### Promotion de la version la plus récente et fonctionnelle

Nous cherchons sur l'interface MLflow notre dernière expérience en passant par `streamflow`, ensuite le dernier run en date.

Nous localisons le modèle `streamflow_churn` et cliquons pour le promouvoir vers le stage `Production`. Son numéro de version est `Version 3` (En effet, j'ai relancé 2 fois le modèle car pas tout à fait correct)

Nous pouvons vérifier que c'est le seul en `Production` en checkant le `Registry Model`

![registry_model](./images_tp4/Capture%20d’écran%202025-12-17%20141319.png)

### Importance de l'interface

La promotion via l'interface est préférable par rapport à un déploiement manuel car elle permet la **traçabilité** avec chaques promotions + timestamp. Elle facilite le **rollback** puisqu'elle répertorie toutes les versions existantes et qu'il suffit permuter les stages de 2 des versions pour revenir en arrière. Il est aussi possible de promouvoir en `Staging` avant de promouvoir en `Production` pour avoir un espace tampon et prevenir les bugs en production. Aussi, j'imagine que l'interface présente l'avantage de faciliter l'intégration CI/CD.


## Exercice 4 : Étendre l’API pour exposer /predict (serving minimal end-to-end)

L'objectif, ici, est de passer d’un endpoint “features” à un vrai endpoint de prédiction en améliorant notre API pour qu'elle récupère le modèle en production.

### Création du endpoint `predict`

Nous modifions pour cela les `requirements.txt` de l'api et ajoutons `MLFLOW_TRACKING_URI`=http://mlflow:5000 dans .env.

Nous ajoutons ensuite dans `api/app.py` un endpoint `POST /predict` et permettons le chargement de `FeatureStore` ainsi que du modèle Mlflow en `Production` et nous finissons en redémarrant l'API.

### Test de l'API et des prédictions

Nous testons l'API en passant par Swagger UI
![predict](./images_tp4/Capture%20d’écran%202025-12-17%20144811.png)

La requête a bien fonctionné et la réponse est la suivante :
```json
{
  "user_id": "7590-VHVEG",
  "prediction": 1,
  "features_used": {
    "plan_stream_tv": false,
    "monthly_fee": 29.850000381469727,
    "months_active": 1,
    "plan_stream_movies": false,
    "net_service": "DSL",
    "paperless_billing": true,
    "watch_hours_30d": 24.48365020751953,
    "skips_7d": 4,
    "avg_session_mins_7d": 29.14104461669922,
    "rebuffer_events_7d": 1,
    "unique_devices_30d": 3,
    "failed_payments_90d": 1,
    "ticket_avg_resolution_hrs_90d": 16,
    "support_tickets_90d": 0
  }
}
```
Le modèle prédit un churn pour cet utilisateur.
Par ailleurs, le modèle dans l'api point vers `models:/streamflow_churn/Production` plutôt qu'un fichier local. Cela permet de ne pas dépendre de la version et donc de ne pas avoir à modifier l'API en fonction. Le rollback ou la mise à jour de version devient donc instantané (car pas de dépendance à la version). En plus de tout cela, on évite les risques de **recopie de fichier ou de nom de fichiers compliqués**

## Exercice 5 : Robustesse du serving : cas d’échec réalistes (sans monitoring)

### Tests `user_id` existant et inexistant

Nous exposons le cas d'un `user_id`existant
![predict_something](./images_tp4/Capture%20d’écran%202025-12-17%20144811.png)

```json
{
  "user_id": "7590-VHVEG",
  "prediction": 1,
  "features_used": {
    "plan_stream_tv": false,
    "monthly_fee": 29.850000381469727,
    "months_active": 1,
    "plan_stream_movies": false,
    "net_service": "DSL",
    "paperless_billing": true,
    "watch_hours_30d": 24.48365020751953,
    "skips_7d": 4,
    "avg_session_mins_7d": 29.14104461669922,
    "rebuffer_events_7d": 1,
    "unique_devices_30d": 3,
    "failed_payments_90d": 1,
    "ticket_avg_resolution_hrs_90d": 16,
    "support_tickets_90d": 0
  }
}
```

Nous faisons à présent le test avec un `user_id` non existant.
Voici, ce que nous obtenons :

![predict_nothing](./images_tp4/Capture%20d’écran%202025-12-17%20145916.png)

```json
{
  "error": "Missing features for user_id=7590",
  "missing_features": [
    "plan_stream_tv",
    "monthly_fee",
    "months_active",
    "plan_stream_movies",
    "net_service",
    "paperless_billing",
    "watch_hours_30d",
    "skips_7d",
    "avg_session_mins_7d",
    "rebuffer_events_7d",
    "unique_devices_30d",
    "failed_payments_90d",
    "ticket_avg_resolution_hrs_90d",
    "support_tickets_90d"
  ]
}
```
En effet, le modèle ne connait pas le user et donc ne peut rien prédire.

### Robustesse du serving

**Deux causes principales d'échec**: 
1. **Entité absente** - Le `user_id` n'existe pas dans l'online store Feast → toutes les features sont NULL
2. **Online store obsolète** - Les features ne sont pas à jour (stale) → valeurs manquantes

L'API détecte ces deux cas via le check `X.isnull()` et retourne une erreur explicite avec `missing_features`. Sans cette vérification, le modèle ferait des prédictions silencieuses sur des données incomplètes, ce qui est dangereux en production.


## Exercice 6 : Réflexion de synthèse (ingénierie MLOps)

### Avantages MLflow
MLFlow assure la **traçabilité complète** des entrainements : chaque run est enregistré avec un `run_id` unique, des **métriques** et **artefacts**, permettant d'identifier exactement quel modèle a produit quels résultats. Le `Model Registry` organise les versions et les stages (None/Staging/Production), facilitant ainsi les **rollbacks** : pour revenir à une version antérieure, il suffit de changer son stage en `Production` sans toucher au code.

### Définition Stage Production
Le stage `Production` définit quelle version du modèle l'API utiliser au démarrage via `models:/streamflow_churn/Production`. À chaque requête, l'API charge **automatiquement** la version marquée `Production` sans modification de code. Cela permet les **rollbacks instantanés** : changer de version consiste à relabelliser une autre version en `Production`, sans intervention sur l'API. Côté déploiement, cela **prévient les erreurs manuelles** (mauvais fichier, mauvaise version) et facilite la **CI/CD** en découplant le versionning du code applicatif.

### Problème restant contre la reproducibilité
Même avec le système complet (Feast + Prefect + PostgreSQL + MLflow + Docker), la reproductibilité peut casser si : les données PostgreSQL sont modifiées (pas de snapshot brut), les images Docker utilisent des tags flottants (python:3.10 peut changer silencieusement), ou l'online store Feast diverge de l'offline store (asynchrone).