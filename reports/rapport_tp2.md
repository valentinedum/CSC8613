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

