# Benchmark de latence — german-credit, modèle v9

1000 appels par backend, après 10 appels de chauffe · x86_64 · Python 3.12.15

| Backend | P50 (ms) | P95 (ms) | P99 (ms) |
|---|---|---|---|
| sklearn_pipeline | 17.521 | 36.528 | 60.511 |
| onnx_python | 0.331 | 0.596 | 0.783 |
| onnx_batch_100 | 1.147 | 1.757 | 2.033 |

Accélération ONNX vs scikit-learn (P50, 1 ligne) : x52.9
