# Guía local — E2E (API + fenix-local + projector)

**Última actualización:** 2026-06-25 · **Status:** `plan.md`

| Servicio | Puerto |
|----------|--------|
| Postgres (dataset 250k) | `5432` |
| Postgres (dataset 95k) | `5433` |
| Redis | `6379` |
| fenix-local | `9000` |
| shipments-history-api | `9100` |
| shipments-history-projector | `9200` |

---

## 1. Paso a paso rápido

Copiá este bloque. Completá variables tras el seeder de journeys (paso D).

```bash
export API=http://localhost:9100/api/v1
export PROJ=http://localhost:9200/api/v1
export JOURNEY_ID=
export ENTRY_ORDER=
export LAST_MILE_ID=

# A. Infra (4 terminales): postgres → fenix-local → API → projector
#    Migraciones si DB nueva: npm run db:migrate en API y projector

cd shipments-history-api
npm run db:seed:app-config && npm run db:seed:status-mappings

cd ../shipments-history-projector
npm run db:seed:bootstrap
npm run db:seed:delivery-history

cd ../shipments-history-api

node src/infrastructure/database/seeders/seedEpReadyToCollectOrders.js \
  --base-url=$API --count=20

npm run db:seed:journey-announcements-from-ep-ready
# → anotá journeyId + entryOrderExternalId

curl -s -X POST $API/ingestor/fenix-queue/consume \
  -H 'Content-Type: application/json' \
  -d "{
    \"body\": \"{\\\"eventVersion\\\":1,\\\"eventName\\\":\\\"JOURNEY_RECONCILIATION\\\",\\\"eventUser\\\":\\\"local\\\",\\\"eventTimestamp\\\":\\\"$(date -u +%Y-%m-%dT%H:%M:%S.000Z)\\\",\\\"data\\\":{\\\"journeyId\\\":\\\"$JOURNEY_ID\\\"}}\",
    \"metadata\": { \"queue\": \"shipments-history-api-to-projector\", \"request_id\": \"reconcile-001\", \"key\": \"$JOURNEY_ID\", \"retries\": 0, \"partition\": 0, \"offset\": 0 }
  }"

curl -s -X POST $API/ingestor/fenix-queue/consume \
  -H 'Content-Type: application/json' \
  -d "{
    \"body\": \"{\\\"eventVersion\\\":1,\\\"eventName\\\":\\\"OE_LAUNCH\\\",\\\"eventUser\\\":\\\"local\\\",\\\"eventTimestamp\\\":\\\"$(date -u +%Y-%m-%dT%H:%M:%S.000Z)\\\",\\\"data\\\":{\\\"entryOrderExternalId\\\":\\\"$ENTRY_ORDER\\\"}}\",
    \"metadata\": { \"queue\": \"WM2_RS_MAIN_OE_LAUNCH\", \"request_id\": \"oe-launch-001\", \"key\": \"$ENTRY_ORDER\", \"retries\": 0, \"partition\": 0, \"offset\": 0 }
  }"

curl -s "$PROJ/delivery-history/shipments?limit=3" | jq '.results[] | {shipment, status, currentMilestone}'
curl -s "$PROJ/delivery-history/catalog/milestones" | jq '.items[].value'
# Esperado: WAREHOUSE_RECEIVED, IN_TRANSIT / milestones: 5 keys

# Andreani opcional (--count=1 siempre)
node src/infrastructure/database/seeders/simulateAndreaniEvents.js \
  --base-url=$API --id-andreani=$LAST_MILE_ID --scenario=entregada --count=1
```

| # | OK si… |
|---|--------|
| E | journey `RECONCILED`; fenix log `JOURNEY_RECONCILIATION` |
| F | fenix log `PACKAGE_NEW_MILESTONE` → `:9200` |
| G | `currentMilestone: WAREHOUSE_RECEIVED` |

### `.env` mínimo

- **API:** `APP_PORT=9100`, `CORE_SKIP_FENIX_QUEUE=false`, `FENIX_GENERATOR_URL=http://localhost:9000/api/event`
- **Projector:** `APP_PORT=9200`
- **fenix-local:** ver `.env.example` (subscribers `:9100` / `:9200`)

### Mocks (`MOCKS_ENABLED`)

`npm run db:seed:bootstrap` deja `MOCKS_ENABLED=false` por defecto. List/detail leen `delivery_history_views` cuando mocks están apagados.

```bash
npm run db:seed:bootstrap          # app_config + catalogs
npm run db:seed:delivery-history   # 60 views default; EP000000001N-01 …
# npm run db:seed:delivery-history -- --count=50000
# optional: --batch-size=50 (default 25; avoid 250+ without raising timeout)
```

Para volver a mocks estáticos (solo dev UI sin DB de shipments):

```sql
UPDATE app_config SET value = 'true', type = 'boolean' WHERE key = 'MOCKS_ENABLED';
```

Reiniciar projector tras cambiar `app_config`.

Con `MOCKS_ENABLED=false`, milestones catalog sigue en seed estático en código hasta migración tabla `milestones`.

### Reset DB

```bash
cd shipments-history-api && npm run db:reset && npm run db:migrate
cd ../shipments-history-projector && npm run db:migrate
```

---

## 2. Postgres Docker Desktop — datasets 95k y 250k (search load)

Para benchmarks de `testing-orchestrator` (`npm run search-load`) conviene **dos contenedores Postgres** con el **mismo CPU/RAM**, datos distintos, y **solo uno levantado** a la vez en M1 Pro (evita competir por la VM de Docker Desktop).

No usamos `docker compose` para esto: contenedores creados desde **Docker Desktop** (UI o terminal integrada). Imagen: **`postgres:16.9`** (misma major que el volumen existente; no recrear un volumen 16 con imagen 15).

| Contenedor | Puerto host | Volumen | Datos |
|------------|-------------|---------|--------|
| `postgres-projector-250k` | `5432` | volumen **existente** (el que ya tiene el seed 250k) | ~250k `delivery_history_views` |
| `postgres-projector-95k` | `5433` | volumen **nuevo** (`postgres_data_95k`) | seed fresco `--count=95000` |

Documentación de performance: `search-performance-spec.md`. Correr estos baselines **antes** de refactorizar queries/índices.

### Recursos recomendados (iguales en ambos)

Ajustá según **Settings → Resources** de Docker Desktop (techo global de la VM).

| Parámetro | Valor inicial |
|-----------|----------------|
| CPU | `3`–`4` |
| Memory | `4`–`6` GB |
| `shm-size` | `256m` |

Anotá CPU/RAM en cada run de Artillery para comparar runs.

### 2.1 Recrear el contenedor 250k con más recursos (conservar datos)

Docker Desktop **no permite** cambiar CPU/RAM de un contenedor ya creado. Hay que pararlo, borrar **solo el contenedor** (no el volumen) y recrearlo.

```bash
# Ver contenedor y volumen
docker ps -a --filter name=postgres
docker volume ls | grep postgres

# Parar y borrar SOLO el contenedor (ej. nombre legacy astartes-postgres-dev)
docker stop astartes-postgres-dev
docker rm astartes-postgres-dev

# Recrear — reemplazá NOMBRE_DEL_VOLUMEN_EXISTENTE por el volumen real del paso anterior
docker run -d \
  --name postgres-projector-250k \
  --cpus=4 \
  --memory=6g \
  --shm-size=256m \
  -v NOMBRE_DEL_VOLUMEN_EXISTENTE:/var/lib/postgresql/data \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=postgres \
  -p 5432:5432 \
  postgres:16.9

# Verificar datos
docker exec -it postgres-projector-250k psql -U postgres -d shipments-history-projector \
  -c "SELECT COUNT(*) FROM delivery_history_views;"
```

**UI Docker Desktop (equivalente):** Containers → Stop → Delete (sin borrar volumen) → Images → `postgres:16.9` → Run → Optional settings: nombre `postgres-projector-250k`, puerto `5432:5432`, volumen existente → `/var/lib/postgresql/data`, env `POSTGRES_USER` / `POSTGRES_PASSWORD`, **Resources** CPU/Memory.

### 2.2 Crear contenedor 95k (volumen nuevo)

Con el de 250k **stopped** (recomendado en M1):

```bash
docker volume create postgres_data_95k

docker run -d \
  --name postgres-projector-95k \
  --cpus=4 \
  --memory=6g \
  --shm-size=256m \
  -v postgres_data_95k:/var/lib/postgresql/data \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=postgres \
  -e POSTGRES_DB=shipments-history-projector \
  -p 5433:5432 \
  postgres:16.9
```

Seed **una sola vez** contra `:5433`:

```bash
cd shipments-history-projector
export DB_95K=postgresql://postgres:postgres@localhost:5433/shipments-history-projector

POSTGRES_DATABASE_URL=$DB_95K npm run db:migrate
POSTGRES_DATABASE_URL=$DB_95K npm run db:seed:bootstrap
POSTGRES_DATABASE_URL=$DB_95K npm run db:seed:delivery-history -- --count=95000 --concurrency=4 --batch-size=25
```

### 2.3 Switch por env (un contenedor activo)

Solo levantá el dataset que vas a testear:

```bash
# Benchmark 250k
docker stop postgres-projector-95k 2>/dev/null; docker start postgres-projector-250k

# Benchmark 95k
docker stop postgres-projector-250k 2>/dev/null; docker start postgres-projector-95k
```

**`shipments-history-projector/.env`** — `POSTGRES_DATABASE_URL`:

| Dataset | URL |
|---------|-----|
| 250k | `postgresql://postgres:postgres@localhost:5432/shipments-history-projector` |
| 95k | `postgresql://postgres:postgres@localhost:5433/shipments-history-projector` |

**`testing-orchestrator/.env`** — alinear `PROJECTOR_DATABASE_URL` al mismo puerto.

Reiniciar el **projector** tras cambiar `.env` (pool Prisma al boot). Para search-load no hace falta la API si `MOCKS_ENABLED=false` y solo corrés Artillery.

```bash
cd testing-orchestrator
npm run search-load
```

### 2.4 Qué no hacer

| Acción | Motivo |
|--------|--------|
| `docker volume rm` sobre el volumen del 250k | Pierde el seed |
| Levantar ambos Postgres durante stress spike | Saturación CPU/RAM en M1 |
| Re-seedear el contenedor 250k “para limpiar” | Horas de trabajo perdidas |
| Confiar solo en `p95` si hay `ERR_SOCKET_TIMEOUT` | El p95 se calcula sobre requests que **completaron** |

---

## 3. Referencia ampliada

### Manual Fenix consume

`POST /api/v1/ingestor/fenix-queue/consume`

| Flow | `body.eventName` | `metadata.queue` |
|------|------------------|------------------|
| Reconcile | `JOURNEY_RECONCILIATION` | `shipments-history-api-to-projector` |
| OE_LAUNCH | `OE_LAUNCH` | `WM2_RS_MAIN_OE_LAUNCH` |
| Classification | `ASSIGN_ITEM_TO_CONTAINER` | `WM2_PD_MAIN_CLASSIFICATION` |
| Dispatch | `DISPATCH_CONTAINER_INTENT` / `CONTAINER_PACKAGES_SHIPPED` | `WM2_PD_MAIN_DISPATCH` |
| Andreani | `andreani-events` | `andreani-events` |

Iteraciones reconcile ≈ `ceil(receipt_items / chunkSize) + 1`.

### WMS manual (6.4 / 6.5)

Sin script — curl a consume con payloads JSON de `blocks-api.md` Block 6.4/6.5. Tras shipped: projector `currentMilestone: LAST_MILE_DISPATCHED`.

### Validación SQL

Todas las queries: **`local-testing-sql.md`**.

### Gotchas

| Tema | Detalle |
|------|---------|
| Dos bases | write `shipments-history-api` / read `shipments-history-projector` |
| `--count=1` | Andreani sim default `5` |
| `--base-url` | scripts → `:9100/api/v1`, no `:9000` |
| Reconcile | no usa `inbox_events`; fenix auto-loop tras 1er kick |
| RECONCILED trap | verificar `journey_packages` count, no solo status |
| Draft vs `-01` | `EP…N` draft; `EP…N-01` cohorte journey |

### Docs

- `plan.md` § B.7 · `fenix-local/README.md` · `prompt.md` · `search-performance-spec.md` · `testing-orchestrator/README.md`
