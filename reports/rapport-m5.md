# Rapport — Lab 5 : serving avec KServe

Binôme : Fatma Ben Gabsia · Dataset : german-credit · Modèle : `scoring-model` v9
Seuil de décision servi : **0.3182**

## 1. Résumé

Le modèle `scoring-model` est déployé avec KServe dans le namespace `scoring`.
Le service utilise une image FastAPI avec ONNX Runtime, exposée via Kourier.
L'autoscaling est configuré avec une cible de 200 RPS et un maximum de 5 réplicas.
La version 2.0.0 a été testée en canary à 10 %, puis promue à 100 % du trafic.
Le service est actuellement READY=True.

## 2. Benchmark de latence

Résultats des tests Locust sur `/predict` :

| Run | Requêtes | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Erreurs (%) |
|---|---:|---:|---:|---:|---:|---:|
| run_100 | 18516 | 170.6 | 310 | 1000 | 1700 | 0.06 |
| run_500 | 32746 | 197.2 | 1900 | 4200 | 5600 | 0.07 |
| run_1000 | 30156 | 170.3 | 4700 | 9500 | 12000 | 0.24 |

Point de saturation : la dégradation devient importante à partir de 500 utilisateurs.
À 1000 utilisateurs, le débit n'augmente plus et le P99 atteint 12000 ms.
Le débit maximal observé parmi ces trois tests est de 197.2 RPS avec run_500.

## 3. Scale-up observé

| Run | Pods (max observé) | CPU moyen par pod | Temps pour atteindre ce nombre de pods |
|---|---|---|---|
| run_100 | Non mesuré | Non mesuré | Non mesuré |
| run_500 | Non mesuré | Non mesuré | Non mesuré |
| run_1000 | 2 observés après le test | Non mesuré | Non mesuré |

Pendant l'observation du test de diagnostic à 1000 utilisateurs, deux pods
étaient actifs avec des consommations CPU de 45m et 12m.

Scale-to-zero : le passage à zéro pod a été observé après une période d'inactivité,
mais le délai exact n'a pas été mesuré.
Démarrage à froid observé : environ 13.6 secondes.
Latence à chaud observée : environ 38 ms.

## 4. Analyse canary

- Répartition observée sur 200 requêtes : 180 × 1.0.0 / 20 × 2.0.0.
- Le split observé correspond exactement à la configuration 90/10.
- Décision : promotion de la version 2.0.0 à 100 % du trafic après vérification
  des réponses et du fonctionnement de l'endpoint `/version`.
- La version 2.0.0 est devenue la version de production.

## 5. Points d'amélioration

- Étudier une cible de scaling adaptée à la charge réellement observée.
- Surveiller les latences P95 et P99 pendant les montées en charge.
- Enregistrer systématiquement le nombre de pods, le CPU et le temps de scale-up
  pendant chaque test.
- Évaluer l'augmentation des ressources et le réglage du nombre maximal de réplicas.
- Tester le batching pour améliorer le débit lorsque la charge est élevée.

## 6. Tests complémentaires

Le test run_1000_diagnostic a obtenu 181.7 RPS, avec un P50 de 3900 ms,
un P95 de 15000 ms, un P99 de 18000 ms et 0.04 % d'erreurs.

Le test run_1000_observe a obtenu 155.8 RPS, avec un P50 de 4400 ms,
un P95 de 14000 ms, un P99 de 52000 ms et 0.12 % d'erreurs.

Ces résultats confirment la dégradation de la latence lorsque la charge augmente.
