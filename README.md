# Clasificador de commits

Servicio que recibe el texto de un mensaje de commit y lo clasifica como `feat`, `fix`, `docs`, `test`, `chore` o `refactor`. Tiene dos motores: **eco** (reglas, sin modelo, instantáneo) y **ollama** (un LLM local, `qwen2.5:0.5b`). Cada petición queda registrada en PostgreSQL. Todo está contenerizado con Docker y validado con GitHub Actions.

## Integrantes y perfil de hardware
- Juan Sebastián Ramos Gutiérrez                                                      - Perfil de hardware utilizado: **B** (8 GB de RAM, modelo `qwen2.5:0.5b`)

## Requisitos
- Mínimo 4 GB de RAM (recomendado 8 GB), 15 GB libres en disco y virtualización habilitada.
- Windows 10 (versión 2004 o superior) u 11 con WSL2 y Ubuntu 24.04, o Linux nativo.
- Docker Engine y Docker Compose, Git. Ollama es opcional (solo para el motor `ollama`).

## Instalación
En Windows, primero abra PowerShell como administrador y ejecute `wsl --install -d Ubuntu-24.04`, reinicie y cree su usuario. Luego, dentro de Ubuntu:

````bash
# 1) Clonar el repositorio
git clone https://github.com/Jsebas03/Clasificador-commits.git
cd Clasificador-commits

# 2) Instalar Docker y Ollama (pide la clave de sudo; al terminar cierre y reabra la terminal)
chmod +x setup.sh diagnostico.sh
./setup.sh

# 3) Crear el archivo de variables de entorno
cp .env.example .env

# 4) Levantar la solución
docker compose up -d --build
````

### Motor ollama (opcional)
````bash
sudo mkdir -p /etc/systemd/system/ollama.service.d
printf '[Service]\nEnvironment="OLLAMA_HOST=0.0.0.0"\nEnvironment="OLLAMA_MAX_LOADED_MODELS=1"\nEnvironment="OLLAMA_KEEP_ALIVE=0"\nEnvironment="OLLAMA_NUM_PARALLEL=1"\n' | sudo tee /etc/systemd/system/ollama.service.d/memoria.conf
sudo systemctl daemon-reload && sudo systemctl restart ollama
ollama pull qwen2.5:0.5b
````
`OLLAMA_HOST=0.0.0.0` es necesario para que la API, que corre dentro de Docker, pueda llegar a Ollama.

## Verificación
````bash
docker compose ps
curl http://localhost:8000/health
curl -X POST http://localhost:8000/clasificar -H "Content-Type: application/json" -d '{"texto":"corrige el error de login","motor":"eco"}'
curl "http://localhost:8000/inferencias?limite=5"
````
También se pueden probar los tres endpoints desde http://localhost:8000/docs.

| Método | Ruta | Descripción |
|---|---|---|
| GET | `/health` | Estado del servicio y de la base de datos |
| POST | `/clasificar` | Cuerpo: `{"texto": "...", "motor": "eco"\|"ollama"}` |
| GET | `/inferencias` | Últimas inferencias registradas (`limite`, 20 por defecto) |

## Pruebas
````bash
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
pytest -v
docker run --rm -i --network host grafana/k6 run - < tests/carga/prueba_carga.js
````
Los resultados completos están en `docs/informe-tecnico.md`. La documentación técnica está en `docs/manual-tecnico.md`.

## Solución de problemas

| Error o síntoma | Causa | Solución |
|---|---|---|
| `422 Field required` en `/clasificar` | El cuerpo JSON usó un campo distinto (`commit_message`) en lugar de `texto` | Enviar `{"texto": "...", "motor": "eco"}` |
| `400 Bad Request` en `/clasificar` desde `/docs` | Swagger prellena `motor` con `"string"` y la API solo acepta `eco` u `ollama` | Cambiar el valor de `motor` a `eco` u `ollama` |
| Latencia negativa (-5394 ms) en la caracterización del modelo | `time.time()` usa el reloj de pared, que se ajustó durante la medición | Medir con `time.perf_counter()`, que no retrocede |
| `localhost:8000` no responde tras iniciar `uvicorn` | El servidor se detuvo con `Ctrl+C` o se cerró la ventana | Volver a iniciarlo, o usar `docker compose up -d` para que corra en segundo plano |
| (pendiente: agregar un quinto error real) | | |

## Video de demostración
(Pegar aquí el enlace al video.)
