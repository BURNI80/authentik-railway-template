# Runbook: crear y publicar el template Authentik en Railway

Guia interna para el autor. Pasos para (1) levantar el proyecto, (2) probarlo end-to-end,
(3) publicarlo como template en el marketplace y (4) opcionalmente pedir verificacion de partner.

Referencia de configuracion exacta: [template.json](../template.json) (snapshot fiel del composer).
Version pinneada actual: `ghcr.io/goauthentik/server:2026.5.6`.

> Restricciones del proyecto (AGENTS.md): no subir a GitHub ni usar el CLI de Railway hasta
> que el autor lo ordene. Este runbook se ejecuta solo cuando se de el visto bueno.

---

## 0. Prerequisitos

- Cuenta Railway con acceso al dashboard (dashboard.railway.com).
- Plan Hobby (o Pro) con un poco de credito. La primera prueba cabe en el trial de $5.
- (Opcional) `railway` CLI para tareas auxiliares; no necesario para publicar el template.

## 1. Proyecto base

1. dashboard.railway.com → **New Project** → **Empty**.
2. Nombra el proyecto, ej. `authentik`.

## 2. Base de datos gestionada

1. En el canvas: **Add New → Database → PostgreSQL**. Servicio **Postgres**.
   - No añadir networking publico. Se alcanza por red privada.
2. El servicio expone `PGHOST, PGPORT, PGUSER, PGPASSWORD, PGDATABASE, DATABASE_URL`.

## 3. Servicio authentik-server

1. **Add New → Docker Image**.
2. **Image**: `ghcr.io/goauthentik/server:2026.5.6`
   (jamás `:latest`; ver [pin de versiones](https://docs.railway.com/templates/best-practices)).
3. **Start command**: `/lifecycle/ak server`
4. **Networking**: habilita **Public Networking (HTTP)**. Railway inyecta `PORT`.
   - Anadir variable `PORT=9000` (Authentik escucha en 9000 por defecto).
5. **Healthcheck** (Settings → Deploy): path `/-/health/ready/`, timeout **600s**.
   - Devuelve 200 solo cuando la conexion a Postgres esta lista; el primer boot corre migraciones.
6. **Volumen**: Attach Volume, mount path **`/data`** (min 1 GB).
7. **Variables** (tab Variables). Todas con referencia, sin valores hardcodeados:

   | Variable | Valor |
   | --- | --- |
   | `PORT` | `9000` |
   | `AUTHENTIK_SECRET_KEY` | `${{shared.AUTHENTIK_SECRET_KEY}}` (o `${{secret(64, "abcdef0123456789")}}`, ver paso 5) |
   | `AUTHENTIK_POSTGRESQL__HOST` | `${{Postgres.PGHOST}}` |
   | `AUTHENTIK_POSTGRESQL__PORT` | `${{Postgres.PGPORT}}` |
   | `AUTHENTIK_POSTGRESQL__USER` | `${{Postgres.PGUSER}}` |
   | `AUTHENTIK_POSTGRESQL__PASSWORD` | `${{Postgres.PGPASSWORD}}` |
   | `AUTHENTIK_POSTGRESQL__NAME` | `${{Postgres.PGDATABASE}}` |
   | `AUTHENTIK_WEB__WORKERS` | `1` |
   | `AUTHENTIK_ERROR_REPORTING__ENABLED` | `false` |

8. **Recursos** (Settings → Deploy → Container Size): **1 GB** RAM.

## 4. Servicio authentik-worker

1. **Add New → Docker Image**.
2. **Image**: `ghcr.io/goauthentik/server:2026.5.6`.
3. **Start command**: `/lifecycle/ak worker`.
4. **Networking**: sin networking publico.
5. **Healthcheck**: ninguno (el worker no expone HTTP; Railway lo reinicia con restart policy).
6. **Variables**: las mismas referencias a Postgres que el server + `AUTHENTIK_SECRET_KEY`
   compartida (paso 5). Sin volumen (media la persiste el server).
7. **Recursos**: **512 MB** RAM.

## 5. Secreto compartido (importante)

`AUTHENTIK_SECRET_KEY` debe ser identica en server y worker.

- **Opcion A (preferida)**: usa **Shared Variables** si el composer del template las expone.
  En el proyecto normal se define en Project → Settings → Shared Variables
  `AUTHENTIK_SECRET_KEY = ${{secret(64, "abcdef0123456789")}}`.
  Ambos servicios referencian `${{shared.AUTHENTIK_SECRET_KEY}}`.
  La funcion `secret()` genera un valor aleatorio nuevo en CADA despliegue del template.
- **Opcion B (fallback)**: define el secreto solo en `authentik-server` con
  `${{secret(64, "abcdef0123456789")}}` y en el worker usa
  `AUTHENTIK_SECRET_KEY = ${{authentik-server.AUTHENTIK_SECRET_KEY}}`.

> No uses `${{secret(...)}}` por separado en los dos servicios: generaria dos claves distintas
> y romperia las sesiones.

## 6. Prueba end-to-end

1. Deploy completo y espera a que el server pase el healthcheck (activo).
2. Abre `https://<authentik-server>.up.railway.app`:
   - Redirige a `/if/flow/initial-setup/` → crea la password de `akadmin`.
3. Login en `/if/admin/`.
4. Crea un **Provider OAuth2/OIDC** de prueba + **Application**.
   - Redirect URI de prueba: `http://localhost:9000/if/session-end/<flow-slug>/` o una URI local tuya.
5. Verifica el flujo SSO completo (login externo con OIDC) y que el worker esta activo
   (System → Workers muestra el worker en `healthy`).
6. Captura de pantallas: setup wizard, admin dashboard, workers, topology del proyecto.
7. (Opcional) Prueba local antes de Railway con `docker compose up` (ver `../docker-compose.yml`).

## 7. Generar y publicar el template

1. En el proyecto: **Settings → Generate Template from Project** → **Create Template**.
   Se abre el **composer**.
2. Revisa que el composer capture todo (espejo de `template.json`):
   - 3 servicios (Postgres, authentik-server, authentik-worker).
   - Variables con referencias `${{...}}` y funciones `secret()` (resueltas al desplegar).
   - Volumen `/data`, healthcheck, networking publico del server.
3. Composer → **Overview / Metadata**:
   - **Nombre**: `Authentik`
   - **Categoria**: `Authentication`
   - **Icono**: logo authentik (1:1, fondo transparente). Ejemplo:
     `https://cdn.jsdelivr.net/gh/homarr-labs/dashboard-icons/svg/authentik.svg`
   - **Descripcion corta**: la de `template.json` (`summary`).
   - **README/Overview**: copia y adapta el contenido de `../README.md`
     (secciones H1/H2: Deploy and Host, About, Use Cases, Dependencies, Why, FAQ).
4. **Publish** (workspace → Templates → Publicar). El template queda en el marketplace
   (private/tests mientras tanto: publicar solo cuando esté probado).
5. Copia el **template ID** de la URL (`railway.com/new/template/<ID>`) y actualiza
   `README.md` (boton Deploy on Railway) y `docs/` si procede.

## 8. Boton Deploy on Railway

```md
[![Deploy on Railway](https://railway.com/button.svg)](https://railway.com/new/template/<TEMPLATE_ID>?utm_medium=integration&utm_source=button&utm_campaign=authentik)
```

Reemplaza `<TEMPLATE_ID>` con el ID real tras publicar.

## 9. (Opcional) Verificacion de partner

Si el proyecto es open source y quieres el badge + mejor posicion:
- Consulta [railway.com/partners](https://railway.com/partners) y aplica con el repo publico.
- Mantén el repo en GitHub publico con README, LICENSE (MIT) y enlace al template.

## 10. Earnings / kickbacks

- 15% del consumo de despliegues de terceros via template; +10% extra (25% total) si respondes
  preguntas en la Template Queue: station.railway.com/my-template-queue.
- Retiro en cash opcional desde Account → Earnings.

## Checklist final

- [ ] Imagen pinneada (no `:latest`).
- [ ] Sin credenciales hardcodeadas; secretos con `secret()`.
- [ ] `AUTHENTIK_SECRET_KEY` compartida identica en server/worker.
- [ ] Postgres solo por red privada (`${{Postgres.*}}`).
- [ ] Healthcheck `/-/health/ready/` con timeout 600s en el server.
- [ ] Volumen `/data` en el server.
- [ ] Prueba e2e completa + capturas.
- [ ] Template publicado, boton actualizado en README, coste estimado documentado.

## Desviaciones del brief (para el informe final)

- **Sin Redis**: authentik 2026.5+ movio la cola de tareas (Dramatiq) y cache a PostgreSQL y
  quito Redis del compose oficial. Decision aprobada por el autor. Stack final: Authentik + PostgreSQL.
- **Storage en `/data`** (no `/media`): cambio del path en versiones 2026.x.
- **Worker sin healthcheck**: no tiene listener HTTP; Railway usa restart policy.
- **Setup por wizard OOBE** (no bootstrap): el usuario elige su password `akadmin` en el primer boot.
