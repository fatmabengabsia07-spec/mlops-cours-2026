# Rapport — Lab 5 : serving avec KServe

Binôme : Maram Yahyaoui et Fatma Ben Gabsia · Dataset : german-credit · Modèle : `scoring-model` v9 ·
Seuil de décision servi : 0.3182

## 1. Résumé

Le modèle de scoring est servi par un InferenceService KServe `scoring-model` en mode Knative, à partir de l'image `localhost:5001/scoring-api` du Lab 4 (FastAPI + ONNX Runtime), exposé par la passerelle Kourier sur le port 30080. L'autoscaler de Knative (KPA) pilote entre 0 et 5 pods sur le débit (cible de 200 requêtes/s par pod, soit 140 effectives). La version 2.0.0 a été déployée en canary à 10 %, puis promue à 100 %.

## 2. Benchmark de latence

| Run | Requêtes | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Erreurs (%) |
|---|---|---|---|---|---|---|
| run_100 | 52085 | 428.0 | 9 | 91 | 210 | 0.00 |
| run_500 | 134057 | 723.9 | 350 | 590 | 850 | 0.00 |
| run_1000 | 134568 | 725.6 | 1000 | 2300 | 3200 | 0.09 |

Point de saturation : le débit plafonne autour de 725 requêtes/s dès 500 utilisateurs (724 contre 726 à 1000 utilisateurs), alors que la latence médiane est multipliée par près de 3 (350 puis 1000 ms) et le P99 par près de 4. Au-delà de 500 utilisateurs, les utilisateurs supplémentaires allongent seulement la file d'attente. Les 117 erreurs du run 1000 sont toutes des 502 Bad Gateway (0,09 %), survenues pendant les pics ; une seule erreur du même type est apparue au run 500.
Limites de la mesure : Locust, Kourier et les pods partagent les mêmes 8 cœurs. Locust a signalé un CPU supérieur à 90 % au run 500 (un seul processus) ; le run 1000 a été lancé avec 4 processus Locust, contrairement aux deux premiers. Le plafond mesuré est donc celui de l'ensemble du banc d'essai.

## 3. Scale-up observé

| Run | Pods (max observé) | CPU moyen par pod | Temps pour atteindre ce nombre de pods |
|---|---|---|---|
| run_100 | 2 à 3 | non mesuré | non mesuré |
| run_500 | 5 (maximum) | non mesuré | non mesuré |
| run_1000 | 5 (maximum) | non mesuré | non mesuré |

Le nombre de pods vient des événements Kubernetes (ScalingReplicaSet), car le tableau de bord ne conserve pas d'historique et metrics-server ne répondait pas (pas de CPU). Les montées se font par paliers rapides : au run 1000, le passage de 3 à 4 pods suit celui de 2 à 3 à 2 secondes d'intervalle. Le palier de 5 pods correspond à maxReplicas.
Scale-to-zero : après la fin de la charge, la descente de 5 pods à 0 prend environ 70 s, et le dernier pod disparaît 2 à 3 minutes après l'arrêt du trafic · démarrage à froid (`curl -w`) : 3,63 s pour le premier appel, contre 0,004 s pour le suivant.

## 4. Analyse canary

- Répartition observée sur 200 requêtes : 178 × 1.0.0 / 22 × 2.0.0.
- Le split est cohérent avec 90/10 : l'attendu est 180 / 20, soit un écart de 2 requêtes. L'écart-type d'un tirage binomial de 200 requêtes à 10 % vaut environ 4,2, donc 22 reste dans la variance normale. Les adresses dédiées latest- et prev- répondent bien en 2.0.0 et 1.0.0.
- Décision : promotion à 100 %. La v2 ne diffère de la v1 que par la route /version et la version annoncée : même modèle (version 9) et même seuil (0.3182), donc mêmes prédictions. Aucun comportement anormal n'a été observé. En production, cette décision s'appuierait sur des métriques automatiques (taux d'erreur, latence, distribution des scores), ce que prévoient les Labs 6 et 7.

## 5. Points d'amélioration

- Augmenter maxReplicas ou donner plus de CPU au banc d'essai : le plafond de 725 requêtes/s est proche de 5 pods × 140 requêtes/s (70 % de la cible de 200), mais la machine est partagée avec le générateur de charge.
- Mesurer le CPU par pod (metrics-server) pour savoir si un pod sature avant d'atteindre la cible, et ajuster scaleTarget en conséquence ; essayer scaleMetric concurrency.
- Plusieurs workers uvicorn par pod : le goulot est HTTP, JSON et Python, pas l'inférence ONNX.
- Regrouper les demandes (batching) pour réduire le coût par prédiction.
- Garder minReplicas à 1 aux heures ouvrées pour éviter le démarrage à froid de 3,6 s ; traiter les 502 observés lors des pics (marge de capacité, sondes de disponibilité).

## 6. (Optionnel) Comparatif des serveurs et bonus batch

Non réalisé (étape optionnelle).
