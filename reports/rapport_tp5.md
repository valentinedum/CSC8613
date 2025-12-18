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