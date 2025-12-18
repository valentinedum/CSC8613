# Rapport TP5

## Exercice 1 : Démarrer la stack pour l'observabilité

Nous mettons à jour notre `docker-compose.yml` pour inclure les services prometheus et grafana. 

Nous créons ensuite le fichier `services/prometheus/prometheus.yml`.

Nous rajoutons des répertoires de provisioning Grafana dans notre projet.

Enfin nous redémarrons notre stack docker et vérifions que tous les services sont up.
![docker_start](./images_tp5/Capture%20d’écran%202025-12-18%20092424.png)

Tout est bon. Nous pouvons aussi vérifier que nous avons accès aux interfaces Prometheus, Grafana.

![prometheus](./images_tp5/Capture%20d’écran%202025-12-18%20092916.png)

![grafana](./images_tp5/Capture%20d’écran%202025-12-18%20092932.png)

Leurs conteneurs sont respectievement `streamflow-prometheus` et `streamflow-grafana`. Prometheus utilise api:8000 au lieu de localhost:8000 pour pouvoir récupérer et stocker les chiffres. localhost pointerait vers son conteneur.

## Exercice 2 : Instrumentation de FastAPI avec de métriques Prometheus

Dans cette partie, notre objectif est d'exposer un endpoint `/metrics` et mesurer volume et latence des requêtes.
Nous adaptons donc notre script `app.py` pour inclure les metrics.

Pour voir si on récupère bien les metrics, on lance la commande suivante:
```bash
curl http://localhost:8000/metrics
```

Voici la réponse:

```
# HELP api_requests_total Total number of API requests
# TYPE api_requests_total counter
api_requests_total 0.0
# HELP api_requests_created Total number of API requests
# TYPE api_requests_created gauge
api_requests_created 1.7660500106799808e+09
# HELP api_request_latency_seconds Latency of API requests in seconds
# TYPE api_request_latency_seconds histogram
api_request_latency_seconds_bucket{le="0.005"} 0.0
api_request_latency_seconds_bucket{le="0.01"} 0.0
api_request_latency_seconds_bucket{le="0.025"} 0.0
api_request_latency_seconds_bucket{le="0.05"} 0.0
api_request_latency_seconds_bucket{le="0.075"} 0.0
api_request_latency_seconds_bucket{le="0.1"} 0.0
api_request_latency_seconds_bucket{le="0.25"} 0.0
api_request_latency_seconds_bucket{le="0.5"} 0.0
api_request_latency_seconds_bucket{le="0.75"} 0.0
api_request_latency_seconds_bucket{le="1.0"} 0.0
api_request_latency_seconds_bucket{le="2.5"} 0.0
api_request_latency_seconds_bucket{le="5.0"} 0.0
api_request_latency_seconds_bucket{le="7.5"} 0.0
api_request_latency_seconds_bucket{le="10.0"} 0.0
api_request_latency_seconds_bucket{le="+Inf"} 0.0
api_request_latency_seconds_count 0.0
api_request_latency_seconds_sum 0.0
# HELP api_request_latency_seconds_created Latency of API requests in seconds
# TYPE api_request_latency_seconds_created gauge
api_request_latency_seconds_created 1.7660500106800396e+09
```

J'ai exécuté plusieurs fois la commande suivante:

```bash
curl -X 'POST' \
  'http://localhost:8000/predict' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "user_id": "7590-VHVEG"
}'
```

A présent quand on appelle le endpoint `/metrics`, on a :

```
# HELP api_requests_total Total number of API requests
# TYPE api_requests_total counter
api_requests_total 6.0
# HELP api_requests_created Total number of API requests
# TYPE api_requests_created gauge
api_requests_created 1.7660500106799808e+09
# HELP api_request_latency_seconds Latency of API requests in seconds
# TYPE api_request_latency_seconds histogram
api_request_latency_seconds_bucket{le="0.005"} 0.0
api_request_latency_seconds_bucket{le="0.01"} 0.0
api_request_latency_seconds_bucket{le="0.025"} 0.0
api_request_latency_seconds_bucket{le="0.05"} 0.0
api_request_latency_seconds_bucket{le="0.075"} 4.0
api_request_latency_seconds_bucket{le="0.1"} 5.0
api_request_latency_seconds_bucket{le="0.25"} 5.0
api_request_latency_seconds_bucket{le="0.5"} 6.0
api_request_latency_seconds_bucket{le="0.75"} 6.0
api_request_latency_seconds_bucket{le="1.0"} 6.0
api_request_latency_seconds_bucket{le="2.5"} 6.0
api_request_latency_seconds_bucket{le="5.0"} 6.0
api_request_latency_seconds_bucket{le="7.5"} 6.0
api_request_latency_seconds_bucket{le="10.0"} 6.0
api_request_latency_seconds_bucket{le="+Inf"} 6.0
api_request_latency_seconds_count 6.0
api_request_latency_seconds_sum 0.7068438529968262
# HELP api_request_latency_seconds_created Latency of API requests in seconds
# TYPE api_request_latency_seconds_created gauge
api_request_latency_seconds_created 1.7660500106800396e+09
```

On remarque en effet que `api_requests_total 6.0` alors qu'avant `api_requests_total 0.0` (ce qui parait cohérent). Mais aussi toutes les metrics concernant `api_request_latency_seconds` sont devenues supérieures. En l'occurence, nous avons un Histogram avec `api_request_latency_seconds_bucket`, où nous voyons que la latence était non nul de `le="0.075"` à `le="+Inf"`. Il est beaucoup plus parlant qu'une moyenne qui cacherait les extrêmes. Ici, on calcule des percentiles et donc on mesure l'expérience connue par la plus part des utilisateurs comme dans le meilleur des cas et le pire. Au final `api_request_latency_seconds_sum 0.7068438529968262` alors qu'avant elle était nulle. Tout est normal, vu qu'on a fait appel à l'API

## Exercice 3 : Exploration de Prometheus (Targets, Scrapes, PromQL)

Nous ouvrons l'interface Prometheuse et vérifions que la target API est bien UP.

![prometheus_api](./images_tp5/Capture%20d’écran%202025-12-18%20105441.png)

L'API est, en effet, **UP** et le dernier Scrape date d'environ **6ms**.

Nous ouvrons maintenant l'onglet `Graph` et exécutons : 
- up : qui signale si le conteneur est vivant (1) ou pas (0). On voit bien qu'il a été démarré à une heure cohérente
- api_requests_total : correspond au compteur brut d'appel api. On remqreu en effet qu'il passe successivement de 0 à 6 au cours du temps.
- rate(api_requests_total[5m]) : représente la vitesse (requetes par seconde)

![api_requests_total](./images_tp5/Capture%20d’écran%202025-12-18%20110240.png)

En effet, il y a eu 6 requêtes en environ 5 min donc 300 sec. 6/300sec donne bien `0.02 RPS`

Nous exécutons :
```
rate(api_request_latency_seconds_sum[5m]) / rate(api_request_latency_seconds_count[5m])
```

Cette valeur représente le temps de réponse moyen (latence moyenne) par requête, calculé sur la fenêtre glissante de 5 minutes.


