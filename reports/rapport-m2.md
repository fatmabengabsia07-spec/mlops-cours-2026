# Rapport comparatif — Kubeflow Pipelines vs Airflow (Lab 2)

**Binôme :** Maram Yahyaoui et Fatma Ben Gabsia · **Date :** 1er novembre 2026 ·
**Machine :** Windows 11 + WSL2 (Ubuntu 22.04), 8 CPU, 7,6 Go alloués à Docker ·
**Dataset :** german-credit

## 1. Contexte

Nous avons converti le pipeline DVC du Lab 1 en un pipeline Kubeflow (KFP) et en un DAG Airflow. Six steps (ingest, validate, prepare, train, evaluate, register) traitent german-credit (1000 lignes, 30 % de mauvais payeurs). Chaque step tourne dans un pod kind avec la même image scoring-pipeline:1.0.0, qui contient le code du Lab 1 : seul le transport change. Les deux outils écrivent dans le même MLflow.

## 2. Résultats quantitatifs

| Mesure | KFP | Airflow |
|---|---|---|
| Installation de zéro (min) | non mesuré (1re tentative hors délai, images lentes) | environ 8 (image tirée en 397 s) |
| Temps jusqu'au premier run réussi (min) | 4,8 (289 s) | non mesuré |
| Durée d'un run complet (min) | 2,2 (130 s) | non mesuré de bout en bout |
| Relance avec les mêmes paramètres (min) | 2,2 (3 steps en cache) | pas de cache dans le DAG |
| Hot-reload (min) | 0,01 de compilation, plus l'upload | non mesuré (scan de dags/ jusqu'à 30 s) |
| Lignes de code propres à l'orchestrateur | 283 | 71 |
| Mémoire consommée (Go) | environ 3 | environ 1,5 (estimé avec free -h) |

## 3. Analyse qualitative

### Developer experience

KFP offre la meilleure boucle de développement : le test local (kfp.local) a exécuté cinq steps en 34 s sans cluster et la compilation prend 0,7 s. Ses artefacts sont typés et l'interface affiche leurs métadonnées, comme le md5 des données identique à celui de dvc.lock. Dans Airflow, aucun fichier ne circule : chaque task recopie l'arborescence via S3 (pipelines/stage.py) et XCom ne porte que de petites valeurs. Le DAG est plus court (71 lignes contre 283) mais sans cache.

### Scaling

Les ressources s'expriment de façon équivalente (set_memory_limit, container_resources) avec un pod par step. L'infrastructure diffère : KFP ajoute 14 pods (environ 3 Go) et son installation a dépassé son délai sur notre réseau lent ; Airflow demande PostgreSQL, Redis, scheduler et worker (environ 1,5 Go estimés). Le cache KFP fait passer un run de 289 s à 130 s, mais cet écart inclut aussi le premier téléchargement de l'image.

### Debugging

Les logs d'un step se lisent dans l'interface des deux outils. Nous n'avons pas provoqué d'échec volontaire et ne comparons que nos incidents : côté KFP, un upload refusé en 409 (nom de version déjà utilisé), un déploiement MySQL hors délai et des pods en Unknown après un redémarrage de Docker ; côté Airflow, aucun incident sur le run. Le lineage de KFP est plus direct : le tag kfp_run_id d'un run MLflow renvoie au run KFP.

## 4. Recommandation

Nous choisirions KFP pour une équipe ML sur Kubernetes qui itère souvent sur l'entraînement, car artefacts typés, cache et lineage sont natifs ; il faut un cluster d'au moins 6 Go et un nouveau tag d'image à chaque changement de code. Nous choisirions Airflow pour planifier des workflows variés (cron, nombreuses intégrations) avec une équipe qui le maîtrise déjà, en acceptant de gérer le transport des fichiers sans cache.

## 5. Limites de la comparaison

Un seul poste (WSL2, 7,6 Go), un cluster kind à trois nœuds, une mesure par configuration. Airflow tournait en Docker Compose ; sa durée de bout en bout et son rechargement n'ont pas été mesurés. Les durées KFP mélangent cache et téléchargement de l'image, les mémoires viennent de free -h et docker stats. Ni parallélisme, ni cluster managé, ni charge réelle, ni échec volontaire n'ont été testés.
