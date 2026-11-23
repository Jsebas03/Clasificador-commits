# Manual técnico

## 1. Arquitectura
```
Cliente (navegador / Postman)
        |  HTTP :8000
        v
   API FastAPI  ---- motor eco (reglas, sin modelo)
        |        \--- motor ollama --> Ollama :11434 (qwen2.5:0.5b, en el host)
        v
   PostgreSQL :5432 (contenedor db-ia, volumen pgdata)
```
- **API FastAPI** (contenedor `api-ia`, puerto 8000): recibe la petición, clasifica y registra.
- **Motor eco**: reglas con expresiones regulares; línea base sin consumo de memoria.
- **Motor ollama**: consulta el modelo local por la API REST de Ollama.
- **PostgreSQL 16 Alpine** (contenedor `db-ia`): guarda cada inferencia.

## 2. Seguridad
- **Puertos expuestos**: 8000 (API, necesario para consumir el servicio) y 5432 solo en `127.0.0.1` (para pruebas locales). Ollama (11434) corre en el host.
- **Roles en la base de datos**: `postgres` (administrador, solo para inicializar y respaldar) y `app_ia` (solo SELECT e INSERT sobre `inferencias`; no puede UPDATE, DELETE ni DROP).
- **Secretos**: en el archivo `.env`, excluido del repositorio con `.gitignore` y `.dockerignore`; el repositorio solo incluye `.env.example`.
- **Contenedor sin privilegios**: la API corre con el usuario `appuser`, no como root.
- **Si se filtra una contraseña**: cambiarla con `ALTER ROLE`, actualizar `.env`, reiniciar los servicios, revisar `inferencias` y rotar cualquier credencial reutilizada.

## 3. Endpoints
| Método | Ruta | Entrada | Respuesta | Errores |
|---|---|---|---|---|
| GET | /health | — | `{"estado":"ok","base_datos":"ok"}` | 503 si no hay base de datos |
| POST | /clasificar | `{"texto": str, "motor": "eco"\|"ollama" (opcional)}` | `motor, modelo, entrada, tipo, latencia_ms` | 400 motor inválido; 422 cuerpo inválido |
| GET | /inferencias | `limite` (int, por defecto 20) | Lista de las últimas inferencias | — |

## 4. Modelo de datos (tabla `inferencias`)
| Campo | Tipo | Significado |
|---|---|---|
| id | SERIAL (PK) | Identificador |
| fecha | TIMESTAMP | Momento de la inferencia |
| motor | VARCHAR(20) | `eco` u `ollama` |
| modelo | VARCHAR(120) | `reglas-v1` o el modelo de Ollama |
| entrada | TEXT | Mensaje de commit recibido |
| salida | TEXT | Categoría devuelta |
| latencia_ms | INTEGER | Tiempo de clasificación |

## 5. Respaldo y restauración
El respaldo se hace con `pg_dump` y se restaura con `psql`. El procedimiento fue probado: se creó el respaldo, se vació la tabla `inferencias` (simulación de desastre) y se restauró; el conteo de registros volvió al valor original y la aplicación siguió insertando con el rol `app_ia`.

````bash
# 1) Crear el respaldo
mkdir -p backups
docker compose exec -T db pg_dump -U postgres --clean --if-exists iadb > backups/respaldo_$(date +%F).sql

# 2) Restaurar
cat backups/respaldo_FECHA.sql | docker compose exec -T db psql -U postgres -d iadb
````

- **Periodicidad propuesta**: una vez al día (por ejemplo, de madrugada con una tarea programada `cron`) y siempre antes de cada despliegue o cambio en el esquema de la base de datos.
- **Responsable**: (Juan Sebastián Ramos Gutiérrez).
- **Almacenamiento**: los archivos `.sql` contienen datos de usuarios, por lo que están excluidos del repositorio (`backups/` en `.gitignore`). Deben guardarse en un lugar aparte y con acceso restringido.
- **Verificación**: cada cierto tiempo se debe restaurar un respaldo en un entorno de prueba para confirmar que sirve.

## 6. Decisiones de diseño y limitaciones
- **Dockerfile multi-etapa**: la imagen final no lleva herramientas de compilación, así que es más liviana y segura.
- **Usuario sin privilegios**: limita el daño si el contenedor es comprometido.
- **Motor eco**: línea base de comparación y respaldo si el modelo no está disponible; no consume memoria.
- **Privilegios mínimos en la BD**: la aplicación solo necesita consultar e insertar.
- **Limitaciones**: el modelo de 0.5B parámetros tiene calidad limitada; con `KEEP_ALIVE=0` la primera respuesta es lenta; se guardan textos de usuarios (considerar Ley 1581 de 2012); sin autenticación en la API.
