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

## 2. Plan de pruebas (Semana 4)

| ID | Tipo | Qué se verifica | Resultado esperado | Obtenido | Estado |
|---|---|---|---|---|---|
| P-01 | Funcional | GET /health responde | Código 200 y estado ok | 200 y {"estado":"ok","base_datos":"ok"} (pytest PASSED) | Aprobada |
| P-02 | Funcional | POST /clasificar con motor eco | Código 200 y tipo correcto | 200 y tipo "fix" para "corrige el error de login" (pytest PASSED) | Aprobada |
| P-03 | Funcional | Motor inválido | Código 400 | 400 (pytest PASSED) | Aprobada |
| P-04 | Acceso | Rol app_ia intenta DELETE y DROP TABLE | Error de permisos | DELETE: "permission denied for table inferencias"; DROP: "must be owner of table inferencias" | Aprobada |
| P-05 | Conectividad | La API resuelve el host db | Devuelve una IP interna | 172.18.0.2 | Aprobada |
| P-06 | Disponibilidad | Reinicio del contenedor de BD | La API se recupera sola | Tras reiniciar db y esperar 15 s, /health respondió ok | Aprobada |
| P-07 | Persistencia | down y up conservan los datos | Los registros siguen existiendo | Los 3 registros previos (ids 1, 2 y 3) siguieron presentes | Aprobada |
| P-08 | Carga | 10 usuarios sobre el motor eco (k6, 2 min) | p95 < 800 ms y errores < 5 % | p95 = 46.21 ms; errores = 0.00 % (0 de 646 peticiones); promedio 98.72 ms; mediana 26.5 ms; máximo 11.49 s (un valor atípico) | Aprobada |
| P-09 | Caracterización | 10 inferencias con modelo | Promedio, mediana y p95 | 9 mediciones válidas: promedio 9397 ms; mediana 6822 ms; p95 16693 ms; mín 5521 ms; máx 18437 ms | Aprobada con observación |

### Observaciones sobre las mediciones
- En P-09, la medición 7 resultó en -5394 ms. Una latencia negativa es imposible: se debió a que el código usaba `time.time()`, un reloj de pared que probablemente fue ajustado por el sistema (sincronización de WSL con Windows) durante la prueba. Se excluyó del cálculo y el código se corrigió para usar `time.perf_counter()`, que no retrocede.
- El valor máximo de 11.49 s en P-08 fue una sola petición atípica; la mediana fue de 26.5 ms.
- Calidad del modelo en P-09: qwen2.5:0.5b acertó 4 de 10 mensajes y clasificó 7 de 10 como "feat". El motor eco acertó 10 de 10 en el mismo conjunto, aunque sus reglas se redactaron con palabras de esos mismos mensajes, por lo que la comparación lo favorece.

### Análisis del cuello de botella
Bajo carga (P-08), el servicio, es decir la API y la base de datos, respondió con una mediana de 26.5 ms y un p95 de 46.21 ms, sin errores. Durante la prueba, `docker stats` mostró un uso bajo: la API consumió entre 4 % y 7 % de CPU y unos 40 MiB de memoria, y la base de datos entre 6 % y 8 % de CPU y unos 12 MiB. El motor eco responde en 0 a 2 ms.

Con el modelo (P-09) la mediana fue de 6822 ms, unas 257 veces mayor que la del servicio. Por tanto, más del 99 % del tiempo de una petición con motor ollama se pierde en la inferencia, no en la API ni en la base de datos.

Causas probables:
1. `OLLAMA_KEEP_ALIVE=0` descarga el modelo de memoria al terminar cada respuesta, así que cada petición vuelve a cargarlo desde el disco. La primera medición (18437 ms) fue la más lenta.
2. La inferencia se ejecuta solo en CPU (2 procesadores asignados a WSL) y sin GPU.
3. Los picos de 12302 ms y 16693 ms sugieren competencia por memoria y CPU con otras aplicaciones de Windows durante la prueba.

### Propuestas de mejora
1. **Mantener el modelo cargado** (`OLLAMA_KEEP_ALIVE=5m`) en equipos con memoria suficiente. Elimina la recarga en cada petición, que es una parte importante de los 5 a 7 segundos típicos. Con 8 GB de RAM y un modelo de unos 400 MB es viable; se debe medir de nuevo la latencia para confirmar la mejora.
2. **Agregar una caché para mensajes repetidos**. Si el mismo texto ya fue clasificado, se devuelve el resultado guardado en milisegundos, sin ejecutar el modelo.
3. **Usar el motor eco como filtro previo**. En las pruebas, eco acertó los mensajes con palabras claras (agrega, corrige, documenta) en 0 a 2 ms. Solo cuando eco no encuentra ninguna regla se consulta al modelo, lo que reduce el número de inferencias lentas.
