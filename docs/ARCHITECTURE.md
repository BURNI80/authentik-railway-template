# Authentik on Railway — Arquitectura

Documentacion tecnica del template `authentik-railway-template`. Explica como encajan las piezas,
que variables usa cada servicio, como funciona el networking y el storage, y el detalle del
start command en Railway (el gotcha que mas dolores de cabeza da).

Version pinneada: `ghcr.io/goauthentik/server:2026.5.6`.

---

## 1. Componentes

```
                       Internet
                          │  HTTPS (Railway proxy)
                          ▼
                ┌───────────────────────┐        ┌───────────────────────┐
                │   authentik-server    │        │   authentik-worker    │
                │  PORT=9000 (listen)   │        │   (no public network) │
                │  UI · API · SSO       │        │  background tasks     │
                │  /data volume (1 GB)  │        └───────────┬───────────┘
                └───────────┬───────────┘                    │
                            │  Railway private network       │
                            └───────────┐        ┌───────────┘
                                        ▼        ▼
                                  ┌─────────────────────┐
                                  │  PostgreSQL (managed)│
                                  │  ${{Postgres.*}}    │
                                  └─────────────────────┘
```

| Servicio | Rol | Red | Storage |
| --- | --- | --- | --- |
| `authentik-server` | Web UI, REST API, flujos SSO, healthcheck | Publica (HTTPS) + privada | Volumen `/data` (media, icons, certs) |
| `authentik-worker` | Blueprints, certificados, eventos, tareas programadas (Dramatiq) | Solo privada | Comparte `/data` (media) |
| `Postgres` | Base de datos unica (datos + cola + cache) | Solo privada | Managed por Railway |

> **Sin Redis**: desde 2026.5, la cola de tareas (Dramatiq) y la cache viven en PostgreSQL,
> igual que el `docker-compose.yml` oficial de upstream. Por eso el stack es solo 3 servicios.

## 2. Variables de entorno

### Server (`authentik-server`)

| Variable | Valor | Proposito |
| --- | --- | --- |
| `PORT` | `9000` | Puerto en el que escucha el binario Rust; Railway lo usa para rutear y hacer healthcheck. |
| `AUTHENTIK_SECRET_KEY` | `${{shared.AUTHENTIK_SECRET_KEY}}` | Firma de cookies/sesiones. Generada una vez por deploy con `${{secret(64, ...)}}`. |
| `AUTHENTIK_POSTGRESQL__HOST` | `${{Postgres.PGHOST}}` | Host de Postgres por red privada. |
| `AUTHENTIK_POSTGRESQL__PORT` | `${{Postgres.PGPORT}}` | Puerto de Postgres. |
| `AUTHENTIK_POSTGRESQL__USER` | `${{Postgres.PGUSER}}` | Usuario. |
| `AUTHENTIK_POSTGRESQL__PASSWORD` | `${{Postgres.PGPASSWORD}}` | Password. |
| `AUTHENTIK_POSTGRESQL__NAME` | `${{Postgres.PGDATABASE}}` | Base de datos. |
| `AUTHENTIK_WEB__WORKERS` | `1` | Workers de gunicorn para el backend Python embebido. Bajo para caber en instancias chicas. |
| `AUTHENTIK_ERROR_REPORTING__ENABLED` | `false` | Sin telemetria de errores. Opcional. |

### Worker (`authentik-worker`)

Igual que el server salvo `PORT` y `AUTHENTIK_WEB__WORKERS`: solo las variables de Postgres
y la `AUTHENTIK_SECRET_KEY` compartida (debe ser identica en ambos, o las sesiones se rompen).

### Reglas de las variables

- Todas con referencia `${{...}}` resueltas en tiempo de deploy. Nada hardcodeado.
- El nombre del servicio Postgres debe ser **exactamente** `Postgres` para que `${{Postgres.*}}` resuelva.
- Nunca usar `${{secret(...)}}` de forma independiente en server y worker: se generarian dos
  claves distintas. Se define UNA vez como variable compartida (`shared`) y ambos la referencian.

## 3. Start command (IMPORTANTE)

Authentik arranca con el script `/lifecycle/ak` que hace bootstrap (espera a Postgres, corre
migraciones al adquirir el lock) y luego `exec` del binario Rust. En Docker Compose se pasa
`command: server` (CMD se **anexa** al ENTRYPOINT de la imagen y funciona).

Railway NO hace eso: aplica el start command como **override del ENTRYPOINT en exec form**
(divide el string en palabras y lo usa como `ENTRYPOINT` del contenedor). Por eso:

| Start command en Railway | Que genera | Resultado |
| --- | --- | --- |
| `server` | `ENTRYPOINT ["server"]` | Fallo: `server: not found`, deploy sin logs. |
| `/lifecycle/ak server` | `ENTRYPOINT ["/lifecycle/ak","server"]` | Funciona, pero deja un proceso Python extra (`ak` + bootstrap) que luego `exec` al binario. |
| `/bin/sh -c "exec ak server"` | `ENTRYPOINT ["/bin/sh","-c","exec ak server"]` | **Recomendado y probado.** `exec` sustituye el shell por el proceso final. |

Regla: en Railway usar SIEMPRE `/bin/sh -c "exec ak server"` y `/bin/sh -c "exec ak worker"`.
En Docker Compose local se usa `server`/`worker` a secas.

## 4. Networking

- **Publico**: solo `authentik-server`. Railway crea un dominio HTTPS y enruta al puerto `PORT`.
  El healthcheck HTTP usa `/-/health/ready/`, que devuelve 200 solo cuando la conexion a
  Postgres esta lista; timeout generoso de 600s porque el primer boot corre migraciones.
- **Privado**: Postgres no se expone. Server y worker hablan con el por `${{Postgres.*}}`
  (host interno `${{Postgres.PGHOST}}`, no la URL publica).

## 5. Storage

- `/data` (volume de Railway, min 1 GB) en el server: media, iconos subidos, fondos, certificados.
- El worker tambien monta `/data` para acceder a la misma media.
- Sin volume, los archivos subidos se pierden en cada redeploy.

## 6. App sleeping (serverless)

Railway habilita `sleepApplication` por defecto (V2 runtime). Consecuencias:

- El server duerme tras inactividad y despierta con la primera request (503 -> 200 en segundos).
- El worker tambien duerme: las tareas programadas solo corren mientras algun servicio esta
  despierto. Para procesamiento de fondo "siempre activo", desactivar el sleep en el worker
  (Settings -> Deploy -> App Sleeping), a costa de consumo continuo (~$4-6/mes).

Esto es lo que mantiene el coste del template dentro del credito Hobby ($5) en uso ligero.

## 7. Primer boot

1. Postgres se provisiona y queda healthy.
2. Server y worker arrancan con `ak server`/`ak worker`.
3. Quien adquiera el lock corre `manage.py migrate` (tardara unos minutos la primera vez).
4. El server levanta el binario Rust en `:9000` y el healthcheck `/-/health/ready/` pasa a 200.
5. El usuario abre el dominio, el wizard `/if/flow/initial-setup/` le pide crear el password
   de `akadmin`. No se usan variables `AUTHENTIK_BOOTSTRAP_*` a proposito (seguridad por defecto).

## 8. Coste

| Recurso | Coste |
| --- | --- |
| Server (1 GB RAM, duerme) | ~$0-10/mes segun uso |
| Worker (512 MB RAM, duerme) | ~$0-6/mes segun uso |
| Postgres (instancia minima managed) | ~$3-8/mes |
| Volume (1 GB) | ~$0.15/mes |

Facturacion por segundo solo mientras los servicios estan despiertos. Ver
[railway.com/pricing](https://railway.com/pricing) para tarifas vigentes.

## 9. Referencias

- Upstream: [github.com/goauthentik/authentik](https://github.com/goauthentik/authentik)
- Docs oficiales: [docs.goauthentik.io](https://docs.goauthentik.io)
- `template.json`: snapshot fiel de la configuracion del composer.
- `docker-compose.yml`: stack equivalente para desarrollo local.
