# Rapport TP2

## Exercice 1: Mise en place du projet et du rapport

### Etat initial du dépôt

![etat_depot](./images_tp2/Capture%20d’écran%202025-12-11%20144800.png)

Notre dépôt a la bonne structure: 
- un dossier api
- un dossier reports
- un fichier docker-compose.yml
Nous avons notre fichier rapport_tp2.md qui n'a pas encore été commité.

### Création de l'armature du projet

Pour débuter ce nouveau TP, nous allons créer les répertoires suivants:

```
db/init/
data/seeds/month_000/
data/seeds/month_001/
services/prefect/
reports/    (existe déjà normalement)
```

![architecture_depot](./images_tp2/Capture%20d’écran%202025-12-11%20145715.png)

Ensuite, nous téléchargeons les données de month_000 et month_001 et nous les insérons dans les dossiers correspondants.

![months_download](./images_tp2/Capture%20d’écran%202025-12-11%20150405.png)

## Exercice 2 : Base de données et docker-compose

Nous créons le fichier d'initialisation de notre base de données : ```db/init/001_schema.sql```.

![creation_db](./images_tp2/Capture%20d’écran%202025-12-11%20150949.png)

Nous allons maintenant créer le fichier ```.env``` qui contiendra les variables d’environnement qui seront automatiquement injectées dans les conteneurs Docker.

![creation_env](./images_tp2/Capture%20d’écran%202025-12-11%20151310.png)

Nous allons aussi mettre à jour le docker-compose.yml pour utiliser deux services : un service postgres pour la base de données et un service prefect.

![maj_dockercompose](./images_tp2/Capture%20d’écran%202025-12-11%20151902.png)

Une fois, le docker-compose.yml nous allons démarrer le service postgres :

![postgre_start](./images_tp2/Capture%20d’écran%202025-12-11%20152112.png)

Puis, nous nous connectons à la base et nous vérifions les tables existantes.

![postgre_tables](./images_tp2/Capture%20d’écran%202025-12-11%20152336.png)

Cela correspond en effet aux tables créées dans le ```db/init/001_schema.sql```

```text
streamflow=# \dt
               List of relations
 Schema |       Name       | Type  |   Owner    
--------+------------------+-------+------------
 public | labels           | table | streamflow
 public | payments_agg_90d | table | streamflow
 public | subscriptions    | table | streamflow
 public | support_agg_90d  | table | streamflow
 public | usage_agg_30d    | table | streamflow
 public | users            | table | streamflow
(6 rows)
```

**Description du schéma** :
- ```labels``` : variable cible indiquant si l'utilisateur s'est désabonné (churn) ou non.
- ```users``` : informations et identifiants uniques des utilisateurs.
- ```payments_agg_90d``` : historique sur les échecs de paiement sur 90 jours.
- ```subscriptions``` : détails contractuels, types d'abonnements, options souscrites, publicités et montants facturés.
- ```support_agg_90d``` : résumé des interactions avec le service client (nombre de tickets, temps de résolution) sur une fenêtre de 90 jours.
- ```usage_agg_30d``` : métriques d'activité et d'engagement de l'utilisateur (temps de visionnage, incidents techniques) sur les 30 derniers jours.

## Exercice 3 : Upsert des CSV avec Prefect (month_000)

Nous allons à présent créer notre service Prefect qui va orchestrer notre pipeline d'ingestion.

Pour cela, nous créons le fichier services/prefect/Dockerfile et services/prefect/requirements.txt.

![services_prefect](./images_tp2/Capture%20d’écran%202025-12-11%20162435.png)

Nous allons maintenant créer un premier flow Prefect qui lit les CSV de month_000 et les insère dans PostgreSQL en utilisant un upsert (INSERT ... ON CONFLICT DO UPDATE).
Pour cela, il nous reste à compléter les #TODO.

### Rôle de la fonction `upsert_csv`

Cette fonction permet une ingestion idempotent ("Upsert" : Update ou Insert). Elle charge les données du CSV dans une table temporair, puis exécute une requête SQL `INSERT ... ON CONFLICT` qui garantit que les nouveaux enregistrements sont ajoutés et que les enregistrements existants sont mis à jour sans erreurs.

### Lancement Flow Ingestion

Nous allons maintenant lancé la création de l'image prefect et lancer le flow d'ingestion pour month_000. Pour cela, nous nous assurons que le service postgres est démarré.

![postgres_check](./images_tp2/Capture%20d’écran%202025-12-11%20163835.png)

A présent, nous lançons le service prefect :
```bash
docker compose up -d prefect
docker compose ps
```

![prefect_on](./images_tp2/Capture%20d’écran%202025-12-11%20164620.png)

Nous lançons le flow d'ingestion avec les variables du mois 000:
![flow_ingestion](./images_tp2/Capture%20d’écran%202025-12-11%20165031.png)

et nous vérifions que les données ont bien été insérées:
![ingestion_check](./images_tp2/Capture%20d’écran%202025-12-11%20165455.png)

Les deux tables comprennnent 7043 lignes donc il y a bien eu ingestion. C'est cohérent avec les fichiers csv sources.

```
streamflow=# SELECT COUNT(*) FROM users;
SELECT COUNT(*) FROM subscriptions;
 count 
-------
  7043
(1 row)

 count 
-------
  7043
(1 row)
```

>Il y a donc 7043 clients après month_000.

## Exercice 4 : Validation des données avec Great Expectations

Nous allons maintenant ajouter une étape à l'ingestion afin d'appliquer quelques expectations, puis de faire échouer le flow si la validation ne passe pas.

Pour cela, nous ouvrons `services/prefect/ingest_flow.py` et ajoutons une nouvelle fonction `validate_with_ge`.

### Rôle de la fonction `validate_with_ge`

Cette fonction intègre la bibliothèque **Great Expectations**  dans le pipeline d'ingestion. Elle vérifie que les données ingérées respectent des règles strictes (schéma des colonnes, absence de valeurs négatives, intégrité des clés). Si une validation échoue, le pipeline s'arrête immédiatement, empêchant la propagation de données corrompues vers les étapes ultérieures (Feature Store ou entraînement des modèles).

### Lancement nouveau Flow Ingestion

A présent, nous lançons le nouveau flow d'ingestion.

![new_ingestion](./images_tp2/Capture%20d’écran%202025-12-11%20171307.png)

Notre pipeline s'est exécutée sans aucune erreur. Nos expectations sont donc correctes et cohérentes avec les données.

### Validation des données

Voici les expectations de `usage_agg_30d`:

```python
elif table == "usage_agg_30d":
        # À compléter : expectations pour usage_agg_30d
        # TODO: vérifier que les colonnes correspondent exactement à l'ensemble attendu
        gdf.expect_table_columns_to_match_set([
            "user_id", "watch_hours_30d", "avg_session_mins_7d", "unique_devices_30d", "skips_7d", "rebuffer_events_7d"
        ])

        # TODO: ajouter quelques bornes raisonnables (par ex. >= 0) sur 1–2 colonnes numériques
        gdf.expect_column_values_to_be_between("watch_hours_30d", min_value=0)
        gdf.expect_column_values_to_be_between("avg_session_mins_7d", min_value=0)
```

**Explication du choix des bornes**: Nous avons choisi une borne inférieure stricte de 0 (min_value=0) car ces métriques représentent des durées physiques. Il est  impossible d'avoir un temps de visionnage négatif ou une durée moyenne de session inférieure à zéro. Une valeur négative indiquerait nécessairement un bug dans le calcul en amont (par exemple, une erreur de soustraction d'horodatages) ou une corruption du fichier source.

**Protection du modèle** : Ces règles agissent comme un pare-feu (Quality Gate) pour le modèle de Machine Learning en excluant les données aberrantes, fausses pour éviter de perturber les modèles. 
Cela garantit la fiabilité des prédictions de churn futures.

## Exercice 5 : Snapshots et ingestion month_001

Nous allons maintenant ajouter la création de snapshots temporels pour figer l’état des données à la fin de chaque mois.

Nous ajoutons donc dans notre `ingest_flow.py` une nouvelle fonction `snapshot_month` pour créer ces snapshots.

### Rôle de `snapshot_month`

Cette fonction historise l'état des données à une date précise définie par le paramètre `as_of`. Elle copie les données "live" (qui changent constamment) vers des tables d'archives (snapshots) immuables. Plus tard, il sera possible de reconstruire fidèlement les caractéristiques (features) d'un utilisateur telles qu'elles étaient au moment exact de la prédiction, nous évitant ainsi le "data leakage".

### Lancement Flow Ingestion month_001

Nous lançons une nouvelle ingestion avec le mois 001 et une nouvelle valeur `AS_OF` après avoir vérifié que nos 2 conteneurs postgres et prefect sont lancés.

![new_ingestion_snapshot](./images_tp2/Capture%20d’écran%202025-12-11%20173418.png)

On va se connecter à la base de données pour vérifier que les deux snapshots existent.

```
streamflow=# SELECT COUNT(*) FROM subscriptions_profile_snapshots WHERE as_of = '2024-01-31';
SELECT COUNT(*) FROM subscriptions_profile_snapshots WHERE as_of = '2024-02-29';
 count 
-------
     0
(1 row)

 count 
-------
  7043
(1 row)
```

**Explication**: Le premier snapshot ne contient pas de lignes alors que le deuxieme en contient 7043. En réalité, nous n'avons pas relancé les données de month_000 avec le nouveau flow d'ingestion.

Nous lançons reprenons donc du début, lançons month_000 puis month_001:
```bash
docker compose exec \
  -e SEED_DIR=/data/seeds/month_000 \
  -e AS_OF=2024-01-31 \
  prefect python ingest_flow.py
```

Cette fois ci, nous avons bien 2 snapshots.
```
streamflow=# SELECT COUNT(*) FROM subscriptions_profile_snapshots WHERE as_of = '2024-01-31';
SELECT COUNT(*) FROM subscriptions_profile_snapshots WHERE as_of = '2024-02-29';
 count 
-------
  7043
(1 row)

 count 
-------
  7043
(1 row)
```

En l'occurence, le mois 001, il n'y a pas eu de changements d'utilisateurs. Les fichiers sources contiennent le même nombre de lignes = 7043.

### Pipeline complet

## Synthèse du TP2 : Ingestion et Historisation

### 1. Schéma du Pipeline d'Ingestion
Voici le flux de données mis en place via Prefect :

![flux_donnes](./images_tp2/Capture%20d’écran%202025-12-11%20180415.png)

**Pourquoi on ne travaille pas directement sur les tables live pour entraîner un modèle ?**
Les tables live sont mutables. S'en servir serait non reproductible. Cela change tout le temps et on perdrait l'historique des données.

**Pourquoi les snapshots sont importants pour éviter la data leakage et garantir la reproductibilité temporelle ?**
Les snapshots sont importants pour eviter le data leakage. Ils figent l'état d'un client à une instant t. Sans eux, le modèle verrait les infos futures, ce qui fausserait l'apprentissage (s'il a déjà l'info avant de prédire)

**Reflexion personnelle**
Ce qui a été le plus difficile dans cette histoire d'ingestion, c'est surement d'intégrer la logique de snapshots. Ils sont indispensebles pour avoir un modele stable non corrompu.
La seule erreur rencontrée est celle à la fin de non execution du script month_000 avant le month_001 qui a été corrigée en effaçant les tables et relançant les ingestions des deux mois.
