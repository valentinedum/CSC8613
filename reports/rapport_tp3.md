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


# Récupération offline & online

# Réflexion