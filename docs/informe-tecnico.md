# Informe técnico

## 1. Caracterización del modelo

| Dato | Valor |
|---|---|
| Perfil de hardware | B (8 GB de RAM) |
| RAM total del equipo | 8 GB |
| Modelo y etiqueta | qwen2.5:0.5b |
| Tamaño en disco | 397 MB |
| Latencia de 5 ejecuciones (ms) | 5430, 2020, 1980, 2060, 2320 |
| Latencia promedio | 2762 ms |
| RAM usada durante la inferencia | ~923 MiB |
| Calidad percibida (1 a 5) | 2 - falló los dos casos de prueba: clasificó "corrige el error de login" como feat y no clasificó "agrega pruebas del endpoint de salud", solo repitió el texto |

La primera ejecución es más lenta porque el modelo debe cargarse en memoria.
