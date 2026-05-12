# Docker Compose (Redis, MinIO y stack opcional API + Celery)

Este repositorio define dos modos de uso con el mismo `docker-compose.yml`:

| Modo | Comando | Qué levanta |
|------|-----------|-------------|
| **Solo infra** (por defecto) | `docker compose up -d` | Redis (6379) y MinIO (API 9100, consola 9101). La API y Celery siguen en el host (p. ej. `.\start-all-dev.ps1`). |
| **Stack aplicación** (perfil `app`) | `docker compose --profile app up -d --build` o `.\scripts\docker-app-up.ps1` | Redis, MinIO, **API** (puerto 8000), **Celery worker** y **Celery beat**. |

El perfil `app` no sustituye el modo “solo infra”: quien solo necesita Redis/MinIO para desarrollo en Windows no tiene que activar el perfil.

## Requisitos

- Docker Desktop (Windows/Mac) o Docker Engine + Compose v2.
- Archivo **`.env.develop`** en la raíz del proyecto (obligatorio para la app; ver [CONFIGURACION.md](CONFIGURACION.md)). Los servicios del perfil `app` montan ese archivo en `/app/.env.develop` dentro del contenedor.

## Variables que Compose fuerza en el perfil `app`

Dentro de la red Docker, `localhost` no apunta a Redis ni a MinIO. Por eso, para `api`, `celery-worker` y `celery-beat` se definen (y tienen prioridad sobre el `.env` cargado por `python-dotenv` con `override=False`):

- `REDIS_URL=redis://redis:6379/0`
- `FILE_STORAGE_ENDPOINT=http://minio:9000`

El resto de variables (SQL Server, SAP, MBA3, JWT, etc.) salen de tu `.env.develop`. Si la base de datos corre **en tu máquina Windows** y en el `.env` usas `SQL_SERVER_SERVER=localhost`, desde el contenedor **no funcionará**: usa el hostname/IP real del SQL Server o `host.docker.internal` si aplica a tu red y puerto (p. ej. `host.docker.internal,1433`). El compose añade `extra_hosts: host.docker.internal:host-gateway` para Linux; en Docker Desktop para Windows suele resolverse igualmente.

La API en contenedor debe poder alcanzar **MBA3**, **SAP Service Layer** y **SQL Server** por red (VPN, firewall, DNS). Eso no lo resuelve Docker por sí solo.

## Comandos útiles

```powershell
# Infra mínima (igual que antes)
docker compose up -d

# Stack completo (primera vez o tras cambios en código Python)
.\scripts\docker-app-up.ps1 -Build

# Solo levantar sin rebuild
.\scripts\docker-app-up.ps1

# Ver logs
docker logs -f mba3-api
docker logs -f mba3-celery-worker

# Parar contenedores del proyecto (volúmenes de datos se conservan)
.\scripts\docker-app-down.ps1

# Parar y eliminar también volúmenes (Redis/MinIO/beat schedule)
.\scripts\docker-app-down.ps1 -Volumes
```

## Detalles técnicos

- **Imagen**: se construye con el `Dockerfile` de la raíz (Python 3.11 + ODBC 17). El código se copia en build; `.env.*` no se incluye en la imagen (`.dockerignore`), solo se monta en runtime.
- **Celery worker**: en contenedor Linux se usa `--pool=prefork`. En Windows nativo el repo usa `--pool=solo` (ver `scripts/dev-terminal.ps1`).
- **Celery beat**: el archivo de programación se guarda en el volumen `celerybeat_data` (`--schedule=/var/lib/celery/celerybeat-schedule`) para que sobreviva a reinicios del contenedor.
- **Puerto 8000**: si ya tienes la API en el host con `uvicorn`, habrá conflicto; detén una de las dos instancias.

## URLs presignadas / MinIO y el navegador

El cliente S3 usa `FILE_STORAGE_ENDPOINT=http://minio:9000` dentro de Docker. Las operaciones servidor–MinIO funcionan. Si en el futuro se devolvieran al navegador URLs firmadas con host `minio`, el navegador no las resolvería; en ese caso haría falta un endpoint “público” separado o proxy. Hoy el flujo principal no depende de eso para el día a día.

## Privacidad / CI

Evita ejecutar `docker compose config` en entornos con `.env.develop` real y compartir la salida: Compose expande `env_file` y puede imprimir secretos.
