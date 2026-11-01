# Rapport comparatif — Kubeflow Pipelines vs Airflow (Lab 2)

**Binôme :** Maram Yahyaoui et Fatma Ben Gabsia · **Date :** 1er novembre 2026 · **Dataset :** german-credit

**Machines (les mesures viennent de deux postes différents) :**
- **Maram :** Windows 11 + WSL2 (Ubuntu 22.04), 8 CPU, 7,6 Go alloués à Docker.
- **Fatma :** Windows, 15,8 Go de RAM, WSL2 Ubuntu 24.04 limité à 10 Go (7,7 Go au début), 8 CPU, Docker Desktop.

## 1. Contexte

Nous avons converti le pipeline DVC du Lab 1 en un pipeline Kubeflow (KFP) et en un DAG Airflow. Six steps (ingest, validate, prepare, train, evaluate, register) traitent german-credit (1000 lignes, 30 % de mauvais payeurs). Chaque step tourne dans un pod kind avec la même image scoring-pipeline:1.0.0, qui contient le code du Lab 1 : seul le transport change. Les deux outils écrivent dans le même MLflow (launched_by=kfp ou launched_by=airflow). Chacune de nous a exécuté le lab sur son poste avec sa propre version du code, de même architecture ; le dépôt contient la version de Maram.

## 2. Résultats quantitatifs

| Mesure | KFP (Maram) | KFP (Fatma) | Airflow (Fatma) | Airflow (Maram) |
|---|---|---|---|---|
| Installation de zéro (min) | non mesuré (1re tentative hors délai, images lentes) | plus de 10 (non chronométré) | environ 15 | environ 8 (image tirée en 397 s) |
| Temps jusqu'au premier run réussi | 4,8 min (289 s) | non comparable (pauses de plusieurs heures) | 1 h 43 (installation, écriture du DAG, diagnostic) | non mesuré |
| Durée d'un run complet (min) | 2,2 (130 s) | 7,8 | 4,8 | non mesuré de bout en bout |
| Relance avec les mêmes paramètres (min) | 2,2 (3 steps en cache) | 5,0 (3 tâches en cache) | 3,6 (sans cache) | pas de cache dans le DAG |
| Hot-reload d'un step | 0,01 min de compilation, plus l'upload | non testé | non testé | non mesuré (scan de dags/ jusqu'à 30 s) |
| Lignes de code propres à l'orchestrateur | 283 | 282 | 70 | 71 |
| Mémoire consommée (Go) | environ 3 | environ 2 (écart de free -h) | 2,2 (docker stats, 7 conteneurs) | environ 1,5 (free -h) |

Les écarts entre nos deux postes (par exemple 2,2 min contre 7,8 min pour un run KFP) ne sont pas expliqués avec certitude. Les pistes probables sont l'image déjà présente ou non sur les nœuds, le réseau et la mémoire allouée à WSL, mais nous ne les avons pas isolées.

## 3. Analyse qualitative

### Developer experience

KFP offre la meilleure boucle de développement : le test local (kfp.local) a exécuté cinq steps en 34 s sans cluster et la compilation prend 0,7 s (mesures de Maram). Ses artefacts sont typés et l'interface affiche leurs métadonnées, comme le md5 des données identique à celui de dvc.lock, ainsi que les métriques et la matrice de confusion. En contrepartie, il faut recompiler et uploader une version à chaque changement. Dans Airflow, aucun fichier ne circule : chaque task recopie l'arborescence du run depuis S3 (pipelines/stage.py) et XCom ne porte que l'identifiant du run MLflow. Le DAG est plus court (70 à 71 lignes contre 282 à 283) mais sans cache. L'installation d'Airflow a été simple sur les deux postes ; celle de KFP a été plus laborieuse (images lentes, MySQL bloqué au démarrage).

### Scaling

Les ressources s'expriment de façon équivalente (set_memory_limit, container_resources) avec un pod par step. L'infrastructure diffère : KFP ajoute 14 pods (2 à 3 Go) ; Airflow demande PostgreSQL, Redis, scheduler et worker (1,5 à 2,2 Go selon la mesure). Le cache KFP raccourcit la relance (289 s à 130 s chez Maram, 7,8 à 5,0 min chez Fatma), mais l'écart de Maram inclut aussi le premier téléchargement de l'image. Côté Airflow, l'écart entre 4,8 et 3,6 min vient d'un démarrage à chaud (image présente, bucket créé), pas d'un cache. Le parallélisme n'a pas été testé.

### Debugging

Les logs d'un step se lisent dans l'interface des deux outils et les pods se suivent avec kubectl. Le lineage de KFP est direct : le tag kfp_run_id d'un run MLflow renvoie au run KFP. Nous n'avons pas provoqué d'échec volontaire et ne comparons que nos incidents.
- **KFP :** upload refusé en 409 (nom de version déjà utilisé) ; déploiement MySQL hors délai ; pods en Unknown ou agent de persistance en CrashLoopBackOff après une coupure de Docker Desktop (runs en PENDING, réparé par un redémarrage du déploiement) ; sur le poste de Fatma, le contrôleur ml-pipeline-scheduledworkflow quittait aussitôt avec le code 0 sur un nœud, contourné en le replanifiant sur un autre nœud, cause racine non établie.
- **Airflow :** aucun incident sur les runs, mais sur le poste de Fatma les health checks Docker affichaient unhealthy alors que l'API de santé répondait healthy, et les commandes de la CLI (airflow dags list) prenaient environ 31 s dans le conteneur.

## 4. Recommandation

Nous choisirions KFP pour une équipe ML sur Kubernetes qui itère souvent sur l'entraînement, car artefacts typés, cache et lineage sont natifs ; conditions : un cluster d'au moins 6 Go, une installation plus lourde (14 pods) et un nouveau tag d'image à chaque changement de code. Nous choisirions Airflow pour planifier des workflows variés (cron, nombreuses intégrations) ou quand une plateforme Airflow existe déjà ; conditions : gérer le transport des fichiers (S3) et se passer de cache natif. Comme les deux exécutent la même image, on peut passer de l'un à l'autre sans réécrire la logique métier.

## 5. Limites de la comparaison

Deux postes WSL2 différents (7,6 Go et 10 Go), un cluster kind à trois nœuds, une à deux mesures par configuration : les durées sont indicatives et non comparables d'un poste à l'autre. La limite mémoire de WSL de Fatma est passée de 7,7 à 10 Go entre ses premiers runs KFP et les suivants. Les durées KFP mélangent cache et téléchargement de l'image. Le hot-reload n'a pas été chronométré de bout en bout et les installations n'ont pas été mesurées avec précision. KFP est en mode standalone et Airflow en Docker Compose avec Celery, pas avec le KubernetesExecutor. Sur le poste de Fatma, la vérification en direct d'Airflow par le script de contrôle est restée bloquée pendant de longues durées (une commande est restée coincée plus de 25 minutes malgré un délai de 90 s) ; la cause n'est pas établie, et le contrôle a été validé avec Airflow arrêté (18/18), les deux runs du DAG ayant réussi. Ni parallélisme, ni cluster managé, ni charge réelle, ni échec volontaire n'ont été testés. Les mémoires sont des mesures au niveau de WSL ou des conteneurs, pas par processus.
