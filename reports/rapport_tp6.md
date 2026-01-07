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
