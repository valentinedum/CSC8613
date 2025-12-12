# Rapport TP3

# Contexte

## Données

Dans les anciens TP, nous avons mis en place l'ingestion des données, la validation avec Great Expectations et la création de snapshots.

Nous disposons donc dans ce TP de 2 snapshots de données `month_000` et de `month_001` datant respectivement de **2024-01-31** et **2024-02-29**. 

Chacun de ces snapshots contient 6 tables :
- ```users``` : informations et identifiants uniques des utilisateurs.
- ```payments_agg_90d``` : historique sur les échecs de paiement sur 90 jours.
- ```subscriptions``` : détails contractuels, types d'abonnements, options souscrites, publicités et montants facturés.
- ```support_agg_90d``` : résumé des interactions avec le service client (nombre de tickets, temps de résolution) sur une fenêtre de 90 jours.
- ```usage_agg_30d``` : métriques d'activité et d'engagement de l'utilisateur (temps de visionnage, incidents techniques) sur les 30 derniers jours.
- ```labels``` : variable cible indiquant si l'utilisateur s'est désabonné (churn) ou non.

## Objectif

Le but de ce TP est de brancher Feast à nos données, de récupérer des features en mode offline et online, et d' exposer un endpoint API simple utilisant ces features.

# Mise en place de Feast

## Construction de l'architecture du service Feast

Nous allons à présent mettre en place Feast en préparant déjà le service Feast côté code, puis en l’ajoutant à la composition Docker. 

Nous commençons par créer l'architecture du **repo feast**.
![architecture_feast](./images_tp3/Capture%20d’écran%202025-12-12%20090026.png)

Nous complétons le Dockerfile associé :
```Dockerfile
FROM python:3.11-slim

WORKDIR /repo

RUN apt-get update && \
    apt-get install -y --no-install-recommends build-essential libpq-dev && \
    rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# On garde le conteneur "vivant" pour pouvoir exécuter feast via docker compose exec
CMD ["bash", "-lc", "tail -f /dev/null"]
```

Nous définissons nos requirements dans `services/feast_repo/requirements.txt`:
```text
feast==0.56.0
pandas==2.3.3
psycopg2-binary==2.9.11
SQLAlchemy==2.0.36
psycopg==3.2.12
psycopg-pool==3.2.7
```

Enfin nous faisons la configuration minimale de Feature Store pour utiliser PostgreSQL en offline et online store dans `services/feast_repo/repo/feature_store.yaml`:

```yaml
project: streamflow
provider: local
registry: registry.db

offline_store:
  type: postgres
  host: postgres
  port: 5432
  database: streamflow
  db_schema: public
  user: streamflow
  password: streamflow

online_store:
  type: postgres
  host: postgres
  port: 5432
  database: streamflow
  db_schema: public
  user: streamflow
  password: streamflow

entity_key_serialization_version: 2
```

Nous compléterons les fichiers `entities.py`, `data_sources.py` et `feature_views.py` par la suite.

Nous ajoutons dans le docker_compose de quoi créer notre service feast.
```yml
services:
  postgres:
    image: postgres:16
    env_file: .env          # Utiliser les variables définies dans .env
    volumes:
      - ./db/init:/docker-entrypoint-initdb.d   # Monter les scripts d'init
      - pgdata:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  prefect:
    build: ./services/prefect
    depends_on:
      - postgres
    env_file: .env          # Réutiliser les mêmes identifiants Postgres
    environment:
      PREFECT_API_URL: http://0.0.0.0:4200/api
      PREFECT_UI_URL: http://0.0.0.0:4200
      PREFECT_LOGGING_LEVEL: INFO
      POSTGRES_HOST: postgres
      POSTGRES_PORT: 5432
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    volumes:
      - ./services/prefect:/opt/prefect/flows
      - ./data:/data:ro     # Rendre les CSV accessibles au conteneur Prefect
  feast:
    build: ./services/feast_repo      # TODO: donnez le chemin de build
    depends_on:
      - postgres
    environment:
      FEAST_USAGE: "False"
    volumes:
      - ./services/feast_repo/repo:/repo # TODO: monter le dossier ./services/feast_repo/repo dans /repo
    

volumes:
  pgdata:
```

## Démarrage du service Feast

Puis nous construisons nos images et démarrons les services en arrière plan.

```bash
docker compose up -d --build
```

**feast** s'est bien lancé.

![build_feast](./images_tp3/Capture%20d’écran%202025-12-12%20093714.png)

### Rôle du conteneur Feast

Feast est un **Feature Store** qui sert de pont entre les données brutes et les modèles, garantissant que les caractéristiques (features) sont identiques lors de l'entraînement (Offline) et de l'inférence (Online). La configuration se trouve dans `/repo`. Nous utiliserons `docker compose exec feast ...` pour enregistrer les définitions avec `feast apply` et charger les données temps réel avec `feast materialize`.


# Définition du Feature Store

## Création de l'Entity

Nous allons maintenant définir notre Feature Store en commençant par définir **l'Entity user**:

```python
from feast import Entity

# TODO: définir l'entité principale "user"
user = Entity(
    name="user",               # TODO
    join_keys=["user_id"],        # TODO
    description="Utilisateur (client StreamFlox)",        # TODO (en français)
)
```

**Definition de l'Entity** : Une entity représente l'objet métier fondamental, ici le client, auquel les features sont rattachées. Elle agit comme une clé qui permet à Feast d'assembler les données de plusieurs sources correctement.

**Choix de `user_id`**
`user_id` est utilisé comme clé de jointure car c'est l'identifiant unique qui relie toutes nos tables PostgreSQL. Chaque feature va donc être relié au bon client lorsqu'on constituera le dataset.

## Définition des Datasources pour les snapshots

Nous remplissone le fichier `services/feast_repo/repo/data_sources.py` pour définir 4 sources de données correspondant à nos 4 tables de snapshots PostgreSQL comme par exemple:

la table **`usage_agg_30d_snapshots`**
qui contient les features suivantes (en plus d'autres):
* `watch_hours_30d` : Nombre total d'heures de visionnage sur le 30 derniers jours.
* `avg_session_mins_7d` : Durée moyenne d'une session sur les 7 derniers jours.
* `rebuffer_events_7d` : Nombre d'incidents techniques (buffering) rencontrés sur les 7 derniers jours.

## Définition des FeatureViews

A present nous allons créer nos **FeatureViews** qui regroupent les features par entité et source. Pour cela, nous créerons 4 vues :
- `subs_profile_fv` : profil d’abonnement 
- `usage_agg_30d_fv` : usage de la plateforme 
- `payments_agg_90d_fv` : paiements récents 
- `support_agg_90d_fv` : interactions avec le support

Pour cela, nous allons faire un script `services/feast_repo/repo/feature_views.py` et definir tous les Field de nos FeaturdViews.

Une fois nos FeatureViews complétées, nous exéutons le conteneur Feast.
![FeatureViews](./images_tp3/Capture%20d’écran%202025-12-12%20105307.png)

Les vues se sont bien déployées sans erreur (mais avec quelques dépreciations).
Le fichier `registry.db` est apparu:

```bash
tree ./services/feast_repo/repo/
./services/feast_repo/repo/
├── __init__.py
├── data_sources.py
├── entities.py
├── feature_store.yaml
├── feature_views.py
└── registry.db

1 directory, 6 files
```

## Rôle de `feast apply`:
C'est la commande de déploiement de Feast. Elle scanne les fichiers Python (`.py`) du dépôt pour détecter les nouvelles *Feature Views* et *Entities*, puis met à jour le registre central (`registry.db`). Elle prépare également l'infrastructure nécessaire (création des tables dans PostgreSQL) pour que les données puissent être stockées et interrogées.

# Récupération offline & online

Dans cette partie nous allons utiliser les features offline et online.

1. Nous créons un répertoire `data/processed`.
2. Nous changeons le `docker-compose.yml` pour lui donner l'accés à Feast et permettre le volume en écriture.
3. Nous redémarrons les services `docker compose up -d --build`
4. Nous créons une nouveau script Python `services/prefect/build_training_dataset.py` pour créer notre dataset à partir des certaines features en joignant nos vues sur le (`user_id`, `event_timestamp`). Le résultat est sauvegardé dans `/data/processed/training_df.csv`

## Lancement du build du dataset

Nou exécutons :
```bash
docker compose up -d
docker compose exec prefect python build_training_dataset.py
```

![build_dataset](./images_tp3/Capture%20d’écran%202025-12-12%20111452.png)

Nous pouvons vérifier qu'il y a bien eu exécution du script de build python et que les données ont bien été écrites dans `data/processed/training_df.csv`

```bash
head -5 data/processed/training_df.csv
user_id,event_timestamp,months_active,monthly_fee,paperless_billing,watch_hours_30d,avg_session_mins_7d,failed_payments_90d,churn_label
7590-VHVEG,2024-01-31,1,29.85,True,24.4836507667874,29.141044640845102,1,True
5575-GNVDE,2024-01-31,34,56.95,False,30.0362276875424,29.141044640845102,0,False
3668-QPYBK,2024-01-31,2,53.85,True,26.7068107231889,29.141044640845102,1,False
7795-CFOCW,2024-01-31,45,42.300000000000004,False,21.8920408062136,29.141044640845102,1,True
```

## Feast temporal correctness

Feast assure la cohérence temporelle en réalisant une jointure point in time entre la date demandée dans le `entity_df` (`event_timestamp`) et la colonne temporelle des sources de données (`timestamp_field="as_of"`). Pour chaque ligne du dataset, Feast sélectionne uniquement la version des données valide à cet instant précis (ici le 31 janvier), en ignorant toutes les mises à jour ultérieures. Ce mécanisme protège le modèle contre le **data leakage** en lui interdisant l'accès aux informations du futur.

## Matérialisation des features dans le Online Store

Nou lançons une commande pour matérialiser nos features dans le Online Store:
`docker compose exec feast feast materialize 2024-01-01T00:00:00 2024-02-01T00:00:00`

Puis nous allons créer un petit script `services/feast_repo/repo/debug_online_features.py` pour tester la fonction `get_online_features` sur un `user_id` existant : "7590-VHVEG"

Nous executons notre script depuis le conteneur feast, voici le résultat obtenu:
```
Online features for user: 7590-VHVEG
{'user_id': ['7590-VHVEG'], 'months_active': [1], 'monthly_fee': [29.850000381469727], 'paperless_billing': [True]}
```

Si par contre on interroge un `user_id` qui n'existe pas ou dont les données n'ont pas été chargées dans le Online Store , Feast ne lève pas d'erreur. Il renvoie simplement des valeurs None pour chacune des features demandées pour cet utilisateur.
Par exemple:
```
Online features for user: 0871
{'user_id': ['0871'], 'monthly_fee': [None], 'months_active': [None], 'paperless_billing': [None]}
```

## Intégration de Feast dans l'API

Pour intégrer Feast dans l'API, nous devons ajouter un service api connecter au Feature Store dans le `docker-compose.yml` et adapter le `Dockerfile` de l'API ainsi que le `requirements.txt`. On finit par reconstruire et redemarrer l'architecture.

Nous rajoutons à notre `app.py` un endpoint `GET /features/{user_id}` qui apelle get_online_features avec un petit sous-ensemble de features.

## Test de l'API

Nous relançons le `docker compose up -d --build`

![feast_api](./images_tp3/Capture%20d’écran%202025-12-12%20145533.png)

Le endpoint `/health` fonctionne toujours tandis que le nouveaux endpoint `features/user_id` fonctionne aussi.
On retrouve les mêmes résultats que tout à l'heure.

```bash
$ curl http://localhost:8000/health
{"status":"ok"}

$ curl http://localhost:8000/features/7590-VHVEG
{"user_id":"7590-VHVEG","features":{"user_id":"7590-VHVEG","monthly_fee":29.850000381469727,"paperless_billing":true,"months_active":1}}
```

# Réflexion

Le endpoint `/features/{user_id}`, basé sur Feast, nous aide à réduire le *training-serving skew* (écart entre l'apprentissage et la production) car il interroge les mêmes features (à travers les `FeatureViews`) qui ont servies à construire le dataset d'entraînement. Feast permet la définition unique des données pour l'Offline (historique) et l'Online (API) ce qui élimine les possibilités d'incohérences entre chaque métier tout le long de la chaine (data engineer -> data scientist -> ml engineer)