# Rapport TP6

## Exercice 1 : Mise en place du rapport et vérifications de départ

Avant de commencer ce TP, nous allons redémarrer la stack et vérifier que les services principaux sont Up.

![docker_start](./images_tp6/Capture%20d’écran%202026-01-07%20092247.png)

Nous vérfions aussi que nous avons bien un modèle en production en nous rendant sur MlFlow (http://localhost:5000)

![mlfow_production](./images_tp6/Capture%20d’écran%202026-01-07%20093216.png)

Il y a bien une version en production. Nous sommes prêts pour commencer le TP.

## Exercice 2 : Ajouter une logique de décision testable (unit test)

Dans cette partie, nous allons implémenter la logique de tests unitaires.
Nous commençons par créer le fichier `services/prefect/compare_utils.py` et créons le test unitaire associé `tests/unit/test_compare_utils.py`.

Nous lançons ensuite les tests:

```bash
(base) duman@PC-Valentine:~/CSC8613$ pytest -q
..                                                                                                                            [100%]
2 passed in 0.05s
```

Les 2 tests créés sont bien passés. On a extrait une fonction pure pour les tests unitaires puisqu'elle est déterministe ce qui permet de la tester de manière rapide et isolée.

## Exercice 3 : Créer le flow Prefect train_and_compare_flow (train → eval → compare → promote)

Nous allons, ici, créer un flow autonome qui :
- Construit un dataset d’entraînement pour un as_of donné (Feast + labels)
- Entraîne un modèle candidat et logge val_auc dans MLflow
- Évalue le modèle Production sur les mêmes données/split
- Compare val_auc et promeut si amélioration >= delta

Pour cela, nous créons un fichier `services/prefect/train_and_compare_flow.py` et l'éxecutons dans un conteneur prefect.

```bash
[COMPARE] candidate_auc=0.6397 vs prod_auc=0.8335 (delta=0.0100)
[DECISION] skipped
[SUMMARY] as_of=2024-02-29 cand_v=4 cand_auc=0.6397 prod_v=3 prod_auc=0.8335 -> skipped
```

Nous allons vérifier dans Mlflow si une nouvelle version a été promue. Normalement cette version 4 est moins bonne que la version 3. Elle devrait être skipped et donc être au statut None.

![mlflow_compare](./images_tp6/Capture%20d’écran%202026-01-07%20104424.png)

C'est le cas, tout est bon.

**Pourquoi utiliser un delta ?**

On utilise un delta (0.01) pour éviter de promouvoir un modèle pour des gains négligeables dus au bruit statistique, et pour s'assurer que l'amélioration est suffisamment significative pour justifier le changement en production.

## Exercice 4 : Connecter drift → retraining automatique (monitor_flow.py)

Nous allons à présent faire en sorte que le nouvel entrainement se déclenche automatiquement lorsqu'il y a la detection de drift. Pour cela, nous adaptons dans le fichier `services/prefect/monitor_flow.py` la fonction `decide_action` pour appeler `train_and_compare_flow` avec un seuil de 0.02.

Puis, nous testons en exécutant le script dans prefect :

```bash
docker compose exec prefect python monitor_flow.py
```

**Logs du monitoring et réentrainement :**

```bash
[COMPARE] candidate_auc=0.6397 vs prod_auc=0.8335 (delta=0.0100)
[DECISION] skipped
[SUMMARY] as_of=2024-02-29 cand_v=6 cand_auc=0.6397 prod_v=3 prod_auc=0.8335 -> skipped
[Evidently] report_html=/reports/evidently/drift_2024-01-31_vs_2024-02-29.html 
            report_json=/reports/evidently/drift_2024-01-31_vs_2024-02-29.json 
            drift_share=0.06 -> RETRAINING_TRIGGERED drift_share=0.06 >= 0.02 -> skipped
```

Le drift détecté est de 6% (drift_share=0.06), ce qui dépasse le seuil de 2%. Le réentrainement a été déclenché automatiquement et un nouveau modèle candidat (v6) a été créé. Cependant, ce modèle n'a pas été promu car son AUC (0.6397) est inférieur à celui du modèle en production (0.8335).

Le rapport HTML généré montre l'analyse du drift entre les deux périodes :

![evidently](./images_tp6/Capture%20d’écran%202026-01-07%20112254.png)

## Exercice 5 : Redémarrage API pour charger le nouveau modèle Production + test /predict

**Pourquoi redémarrer l'API ?**
Notre API charge le modèle au démarrage (MLflow URI models:/streamflow_churn/Production). Donc si une promotion a eu lieu, il faut redémarrer l’API pour qu’elle recharge la nouvelle version depuis le Model Registry. 


**Test de prédiction :**

```bash
curl -s -X POST "http://localhost:8000/predict" \
  -H "Content-Type: application/json" \
  -d '{"user_id":"7590-VHVEG"}'
```

**Réponse JSON :**

```json
{
  "user_id": "7590-VHVEG",
  "prediction": 1,
  "features_used": {
    "months_active": 1,
    "paperless_billing": true,
    "net_service": false,
    "monthly_fee": 29.85,
    "avg_session_mins_7d": 29.14,
    "skips_7d": 4,
    "unique_devices_30d": 2,
    ...
  }
}
```

L'API prédit un **churn (prediction=1)** pour cet utilisateur en utilisant les features récupérées depuis Feast.

## Exercice 6 : CI GitHub Actions (smoke + unit) avec Docker Compose

Nous avons créé un workflow CI dans `.github/workflows/ci.yml` avec deux jobs :

**Job `unit` :**
- Configure Python 3.11
- Installe pytest
- Exécute les tests unitaires

**Job `integration` :**
- Démarre la stack Docker Compose (postgres, feast, mlflow, api)
- Vérifie le healthcheck de l'API
- Upload les logs en cas d'échec

**Pourquoi démarrer Docker Compose dans la CI ?**

On démarre donc Docker Compose dans la CI pour effectuer des tests d'intégration multi-services. Cela permet de vérifier que tous les services (API, PostgreSQL, Feast, MLflow) communiquent correctement entre eux, et que l'API est fonctionnelle dans un environnement proche de la production.

**Résultat du workflow GitHub Actions :**

![github_actions_ci](./images_tp6/Capture%20d’écran%202026-01-07%20114034.png)

Les deux jobs passent avec succès, validant ainsi les tests unitaires et l'intégration de la stack complète.

## Exercice 7 : Synthèse finale : boucle complète drift → retrain → promotion → serving

Ce TP a mis en place une boucle complète de MLOps automatisée qui assure la qualité et la fraîcheur du modèle en production.

**1. Détection du drift (Evidently)**

Le drift est mesuré en comparant les distributions statistiques des features entre deux périodes (référence vs. actuelle). Evidently calcule un `drift_share` qui représente la proportion de features ayant dérivé significativement. 

Le seuil de 0.02 (2%) utilisé dans ce TP est volontairement bas pour forcer le réentrainement dans le cadre du TP. En pratique, ce seuil serait plus élevé (5-10%) pour éviter les réentrainements trop fréquents dus au bruit statistique et limiter les coûts.

**2. Réentrainement automatique (train_and_compare_flow)**

Lorsque `drift_share >= threshold` pour nous à 2%, le flow Prefect `train_and_compare_flow` se déclenche automatiquement et il:
- Construit un dataset d'entraînement avec Feast pour la nouvelle période
- Entraîne un modèle candidat (RandomForest) et logue les métriques dans MLflow
- Évalue le **modèle Production actuel** sur les mêmes données (même split)
- Compare les AUC : `new_auc > prod_auc + delta` (delta=0.01) avec ce petit delta pour éviter la maj à cause du bruit.

La fonction pure `should_promote()` garantit une logique de décision testable et déterministe. Si le candidat est significativement meilleur, il est automatiquement promu en Production via le Model Registry MLflow, et l'ancienne version est archivée.

**3. Déploiement (API restart)**

L'API charge le modèle depuis `models:/streamflow_churn/Production` au démarrage. Après une promotion, un redémarrage (`docker compose restart api`) est nécessaire pour charger la nouvelle version. L'API sert ensuite les prédictions avec les features fraîches récupérées de Feast.

### Rôle de Prefect vs GitHub Actions

Prefect gère les workflows de production comme le monitoring et le réentrainement. Il permet de lancer des tâches de manière automatique, par exemple tous les mois pour vérifier le drift. Il offre aussi une interface pour voir ce qui se passe et relancer en cas d'erreur.

GitHub Actions vérifie que le code fonctionne bien à chaque fois qu'on fait un commit (continuous integration). Il lance les tests unitaires et vérifie que tous les services démarrent correctement avec Docker Compose avec les tests d'intégration. Par contre, il ne fait pas d'entraînement complet car c'est trop long pour la CI.

### Limites et améliorations

**Pourquoi la CI ne doit pas entraîner le modèle**

Entraîner un modèle prend du temps et ralentirait trop la CI. Les résultats changent un peu à chaque fois à cause du hasard dans l'algorithme. La CI sert juste à vérifier que le code est bon, pas à évaluer le modèle.

**Tests manquants**

Il manque des tests pour vérifier que Feast récupère bien les bonnes features. Il faudrait aussi tester que l'API répond correctement avec le bon format JSON. On devrait avoir des alertes si les performances du modèle baissent trop. Aussi, des tests de charge permettraient de voir si l'API tient sous beaucoup de requêtes. 

**Approbation humaine**

En réalité, on ne peut pas tout automatiser. Il faudrait qu'une personne valide avant de mettre un nouveau modèle en production. On devrait aussi surveiller les performances après le déploiement avec par exemple un test A/B ou d'autres type. Il faut garder une trace de qui a mis quel modèle en production et pouvoir faire un rollback si ça se passe mal. Dans des domaines sensibles comme la banque ou la santé, l'approbation humaine est nécessaire.

