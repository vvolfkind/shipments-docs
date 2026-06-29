# Plan.md - Current V1 Completion Plan

Operational plan for `shipments-history-api/`, `shipments-history-projector/`, `fenix-local`.

**Única fuente de status e implementación.** Dominio: `domain-model.md`. Onboarding: `prompt.md`.

## Doc map

| Doc | Rol |
|-----|-----|
| **`plan.md`** | **Único status** + checklist pendiente |
| `prompt.md` | Reglas agente + mapa mínimo (sin status) |
| `domain-model.md` | Dominio (milestones, tramos, gates, watermark) |
| `local-testing-guide.md` | Runbook E2E, SQL validación, search load |
| `andreaniEvents.md` | Production Andreani wire samples (reference; not exhaustive) |
| `testing-orchestrator/README.md` | Pipeline `warehouse` / `e2e` / Artillery |
| `blocks-api.md` / `blocks-projector.md` | Spec diseño por block — **no** usar marcadores Status internos; ver `plan.md` |
| `search-performance-spec.md` | Solo perf lectura search (SP.1 / SP.2) |

Contratos FE (list search, detalle) — entregados a equipos; no viven en este repo.

**Last updated (2026-06-27):** warehouse MVP **coded**. B.5.9 ✅. B.5.10 orchestrator ✅ (pending user E2E validation). SP.1 coded (re-measure pending). See **§ Pending** (B.5.11, SP.1 gate).

---

## Pendiente (v1 — foco implementación)

| ID | Qué | Dónde |
|----|-----|--------|
| **SP.1** | Search read perf (single query, partial indexes) — P1.1–P1.3 coded; **re-run Artillery gate @ 250k** and archive summary | projector · `search-performance-spec.md` | **IN PROGRESS** |
| **C.4** | `PackageEvent.shipmentId` = tramo activo; **`segments[]`** hoy ≈1 (FIRST_MILE no proyecta) | projector mapper |
| **C.2** | Tabla `milestones` + seeder (HTTP catálogo ✅ static) | projector |
| **B.7.4** | `.env.example` + docs locales bulk Fenix | API, `fenix-local/` |
| **B.6** | Runbook manual EP-ready → WMI (orchestrator automatizado ✅) | `local-testing-guide.md` |
| **B.5.11** | **Analysis (debt):** `Visita` + text-only `"Entregado"` in desc fields (no `motivo` 99) — trace legacy TMS/Delivery Manager interpretation before mapping change | ops + API | **OPEN** |
| **B.2.5–B.2.6** | Tests retry/idempotency watermark | API |
| **D.1** | `POST .../reconcile/chunk` | API |
| **D.2** | Script reconciliation loop local | tooling |
| **D.3** | Alinear `.env.example` restante | ambos proyectos |

**Fuera de scope v1 activo:** cancel manual (**B.8.5**), `milestoneKey: FAILED` como terminal (fallos OOLL **proyectan** con macro `FAILED`, no congelan ciclo), SP.2 / Block 13, UNMATCHED replay v2, Block 9 externos.

**Diseño operativo:** terminales última milla → **§ B.8** · Fenix bulk → **§ B.7**

---

## Hecho (warehouse MVP v1)

API: tramos, gates, watermark, bulk LPC, OOLL Andreani. Projector: proyección, search, detalle R2. E2E orchestrator + Artillery. Detalle por fase en tablas A–D abajo.

---

## Implementation Order (executable)

### Fase A — Tramos API (prerequisite)

**Goal:** Correct `FIRST_MILE` / `LAST_MILE` boundaries per `domain-model.md` §3.

| ID | Task | Files (indicative) | Status |
|----|------|-------------------|--------|
| A.1 | Block 2: create `Shipment FIRST_MILE` + `PackageShipment` on ep-ready | `consumeEpReadyToCollectOrders.useCase.ts`, `repositories/shipment/` | **DONE** |
| A.2 | Block 3: remove `ensureLastMileShipmentLink` from chunk; keep `LAST_MILE` identifier link only | `processJourneyReceiptChunk.useCase.ts`, `journey.repository.ts` | **DONE** |
| A.3 | OE_LAUNCH: close first mile (`exitedAt`), open `LAST_MILE`, include first-mile timestamps in payload | `consumeOeLaunch.useCase.ts`, `oeLaunch.repository.ts`, `shipment.repository.ts` | **DONE** |
| A.4 | Unit tests for tramo boundaries | `consumeOeLaunch*.spec.ts`, `processJourneyReceiptChunk*.spec.ts`, `consumeEpReady*` | **DONE** |

**Exit criteria:** ✅ Met in code; validate on DB during B.6 E2E.

---

### Fase B — Gates & emission contract

**Goal:** Align `PACKAGE_NEW_MILESTONE` with `domain-model.md` §4–§5.

| ID | Task | Files (indicative) | Status |
|----|------|-------------------|--------|
| B.1 | Bootstrap + dispatch gate payloads: `milestoneKey`, `IN_TRANSIT`, `operationalTimestamps`, `trailTransitions` / `warehouseContext` | `consumeOeLaunch.useCase.ts`, `consumeWmsContainerPackagesShipped.useCase.ts` | **DONE** (OE_LAUNCH + CONTAINER_PACKAGES_SHIPPED) |
| B.2 | Projection watermark — lectura en gates + claim en producer | Ver subtasks abajo | **DONE** (B.2.3–B.2.4); B.2.5–B.2.6 pending |
| B.3 | Block 6.4 classification: persist only, no outbox | `consumeWmsPackageClassification.useCase.ts` | **DONE** |
| B.4 | Block 6.5 dispatch: gate `LAST_MILE_DISPATCHED` + bulk transactional apply | `consumeWmsContainerPackagesShipped.useCase.ts`, `wmsDispatch.repository.ts` | **DONE** |
| B.5 | Block 5 Andreani: MATCH → incremental OOLL + `DELIVERED`; UNMATCHED persistido (forward-only, replay v2) | `consumeAndreaniEvents.useCase.ts`, `repositories/andreani/`, `oollProcessing/` | **DONE (MVP v1)** — B.5.8 ✅ · **B.5.9** ✅ |
| B.6 | E2E via `fenix-local`: EP-ready → WMI → reconcile → OE_LAUNCH → classification → dispatch → projector | `testing-orchestrator` + `local-testing-guide.md` | **PARTIAL** — orchestrator `warehouse` ✅ `2026-06-25T16-59-55-015Z`; `e2e` ✅ `2026-06-25T16-33-08-710Z`; runbook manual EP-ready → WMI pending |
| B.7 | Fenix bulk relay: outbox masivo → chunks secuenciales → `produceBulk` | `fenix.producer.ts`, `fenix.outbox.producer.ts` | **DONE (core)** — B.7.4 docs pending |

**Exit criteria:** API emits gates 1–2 with `milestoneKey`; classification does not emit; dispatch emits gate 2 — **coded**; watermark lectura en gates + claim en producer ✅ **B.2.3–B.2.4**; projector `current_milestone` ✅ **C.1 + C.3**; orchestrator E2E ✅ (`warehouse` + `e2e`); runbook manual B.6 pending.

**B.2 design (closed — `domain-model.md` §5.1):**

| Subtask | Qué | Dónde | Status |
|---------|-----|-------|--------|
| B.2.1 | Schema `package_projection_watermarks` | `schema.prisma` | **DONE** |
| B.2.2 | Repo watermark (`findByPackageIds`, `claimBulk`) + entity | `repositories/packageProjectionWatermark/` | **DONE** |
| B.2.2b | Servicio delta read-path (`filterCandidateAgainstWatermark`, `buildClaimFromDelta`) | `application/packageProjectionWatermark/packageProjectionDelta.service.ts` | **DONE** |
| B.2.3 | Integrar lectura watermark + filtro delta en gates | `consumeOeLaunch.useCase.ts`, `consumeWmsContainerPackagesShipped.useCase.ts`, `consumeAndreaniEvents.useCase.ts` | **DONE** |
| B.2.4 | Tx claim: upsert watermark + `createManyAndReturn` outbox `PENDING` | `framework/.../fenix.outbox.producer.ts`, `PackageProjectionOutboxCoordinator` | **DONE** |
| B.2.5 | Relay retry con `eventId` sin re-claim watermark | producer path sequential (existente) | **NEXT** |
| B.2.6 | Tests integración: gate replay sin `event`, producer idempotent claim | `test/unit/ingestor/*`, `fenixOutboxProducer.spec.ts` | **PARTIAL** — gate replay + producer tx tests ✅; idempotent claim **B.2.6** |

**B.7 Fenix bulk relay (single source de diseño + status):**

**Problema (resuelto B.7.2):** gates LPC masivos (`OE_LAUNCH`, `CONTAINER_PACKAGES_SHIPPED`) devuelven `EventResponse[]` (decenas–cientos). Outbox + watermark bulk ✅ (B.2.4). Antes: relay Fenix = **1× `POST /api/event` por fila** → timeouts E2E. Ahora: **`POST /api/event/bulk`** en chunks de `FENIX_BULK_CHUNK_SIZE` (**B.7.2** ✅). Evidencia orchestrator: `warehouse` `2026-06-25T16-59-55-015Z`.

**Alcance LPC:** solo cohortes masivas con `publishManyEventsBulk`. Andreani incremental (1 evento) y retries con `payload.eventId` siguen en `produce()` secuencial.

**Contrato Fenix (generator — ref. `fenix-local/`, `fenix-queues/queue-event-generator`):**

| Endpoint | Uso |
|----------|-----|
| `POST /api/event` | Single / retry |
| `POST /api/event/bulk` | Cohort — body `events_with_key[]` (**XOR** `events[]`) |

```json
{
  "producer": "shipments-history-api",
  "queue": "shipments-history-api-to-projector",
  "events_with_key": [
    { "key": "<package externalId>", "event": "<NotificationMessage JSON string>" }
  ]
}
```

| HTTP | Body | Acción API |
|------|------|------------|
| `200` | `OK` | Chunk OK → outbox rows `SENT` |
| `207` | `failed to deliver N messages` | Fallar **chunk entero** → rows `FAILED` (generator no indica cuáles) |
| `500` / red | ApiError / error | Fallar chunk entero → `FAILED` |

**Límites:** 1 MiB **por** string `event` (Kafka default); generator `LimitSendMessagesBulk` = **100** por `SendMessages`. LPC hoy ~1.5–3.5 KB/mensaje → N ≈ **100** (`FENIX_BULK_CHUNK_SIZE`), no el MiB. `produceBulk` valida tamaño por mensaje (`FENIX_MAX_MESSAGE_BYTES`).

**Patrón relay B.7.2+ (obligatorio — secuencial, sin `Promise.all` entre chunks):**

```text
1. tx: watermark claim + createManyAndReturn outbox (PENDING)     ← ya existe (B.2.4)
2. preparedEvents + createdRows en memoria (no re-query outbox)
3. for each chunk of N (alineado outbox ids ↔ payloads):
     produceBulk(chunk)  →  events_with_key[].key = event.identifier
     updateMany outbox: chunk OK → SENT | chunk fail → FAILED (todo el chunk)
4. siguiente chunk solo tras cerrar estado del anterior
```

**Re-emit / dedup:** mismo `externalEventId` determinístico + outbox retry; projector dedupea `(source, externalEventId)` → re-envío de chunk fallido es seguro para filas ya aplicadas.

**Env (`fenix.validation.ts`):**

| Variable | Default | Rol |
|----------|---------|-----|
| `FENIX_GENERATOR_URL` | — | Base single (`…/api/event`) |
| `FENIX_GENERATOR_URL_BULK` | opcional | Override; si ausente → `{FENIX_GENERATOR_URL}/bulk` |
| `FENIX_BULK_ENABLED` | `true` | `false` → loop `produce()` legacy |
| `FENIX_BULK_MIN_SIZE` | `2` | Mínimo eventos para bulk (`shouldUseBulkRelay`) |
| `FENIX_BULK_CHUNK_SIZE` | `100` | N mensajes por chunk HTTP (= generator) |

**No hacer:** `Promise.all` entre chunks; `setTimeout` fijo entre chunks (salvo rate-limit demostrado); bulk en retries con `eventId`; projector bulk consume.

| Subtask | Qué | Dónde | Status |
|---------|-----|-------|--------|
| B.7.0 | Local emulator `/api/event` + `/api/event/bulk` | `fenix-local/` | **DONE** |
| B.7.1 | `FenixQueueProducer.produceBulk` + env + errores 207/500 | `framework/.../fenix.producer.ts` | **DONE** |
| B.7.2 | `publishManyEventsBulk`: outbox masivo → chunks N secuenciales → `produceBulk` + `updateMany` por chunk | `fenix.outbox.producer.ts` | **DONE** |
| B.7.3 | Tests outbox bulk relay (207 → chunk FAILED, `shouldUseBulkRelay`, retry secuencial) | `fenixOutboxProducer.spec.ts` | **DONE** |
| B.7.4 | `.env.example` + docs locales alineados | API, `fenix-local/README.md` | **PARTIAL** |

**Acceptance B.7:** OE_LAUNCH ~50 pkgs → ⌈50/N⌉ bulk HTTP (no 50 singles); keys = `externalId`; retry `eventId` → `/api/event`; fenix-local E2E N `ConsumerRecord`s; `FENIX_BULK_ENABLED=false` → per-event. **Evidencia:** orchestrator `warehouse` `2026-06-25T16-59-55-015Z` (~110s, 198/198).

**Patrón outbox (no romper):** gate use case devuelve `{ event }`; `EventPublisherInterceptor` activa outbox **post-return**. El use case **no** escribe `outbox_ingestor`. Warehouse replay: `appliedPackageIds` + set-once sigue siendo primera línea de defensa.

**B.5 design (closed — `blocks-api.md` §5.8):**

| Subtask | Qué | Dónde | Status |
|---------|-----|-------|--------|
| B.5.1 | Extender `EventStatus` con `UNMATCHED` + índice parcial inbox Andreani | `schema.prisma`, `20260626100000_add_inbox_andreani_unmatched_index` | **DONE** |
| B.5.2 | Lookup package por `idAndreani` (criterio MATCH §5.8.2) | `andreani.repository.ts` → `findMatchByTrackingId` | **DONE** |
| B.5.3 | Integrar `CarrierStatusMapping` + matriz §5.3 | `oollProcessing.service.ts` (+ Andreani facade) | **DONE** |
| B.5.4 | MATCH: persist milestones/events + outbox incremental / gate `DELIVERED` | `andreani.repository.ts` + use case | **DONE** |
| B.5.5 | NO MATCH: inbox `UNMATCHED`, sin dominio/outbox (forward-only) | `consumeAndreaniEvents.useCase.ts` | **DONE** |
| B.5.6 | Tests: MATCH, UNMATCHED, idempotencia, outbox, sin replay v1 | `test/unit/ingestor/consumeAndreaniEvents.useCase.spec.ts` | **PARTIAL** — 6 tests; falta `ambiguous_match` explícito |
| B.5.7 | Replay UNMATCHED | — | **v2 — fuera MVP** |
| B.5.8 | Extract shared `OollProcessingService` (matrix + watermark + outbox); thin per-carrier consumers | **`blocks-api.md` §5.9** | **DONE** — E2E `2026-06-25T16-33-08-710Z` |
| B.5.9 | Watermark failure context delta — re-emitir fallos con motivo/comentario distinto; timeline N×`FAILED` → `DELIVERED` | `packageProjectionDelta.service.ts`, watermark schema, tests, orchestrator extensión | **DONE** — ver § B.5.9 abajo |

**B.5.8 scope (summary — detail in `blocks-api.md` §5.9 only):**

- Move Block 5.3 matrix, watermark delta, listable gate, and `PACKAGE_NEW_MILESTONE` assembly out of `consumeAndreaniEvents.useCase.ts` into `application/oollProcessing/`.
- Andreani keeps: DTO, Fenix handler, MATCH/UNMATCHED, `repositories/andreani/`, raw-key adapter.
- Future operators (OCASA, ClicPaq, MOOVA, …): new consumer + repository + adapter; **reuse** `OollProcessingService`.
- **Mandatory:** rewrite/split unit tests (`oollProcessing.service.spec.ts` + thinned Andreani spec). No behavior change for Andreani (orchestrator E2E must pass unchanged).
- **CRITICAL:** implementing agents **must not run terminal commands** — human/CI runs build, unit tests, and E2E.

**B.5.9 Watermark failure context delta (DONE):**

**Problem solved:** repeat OOLL failures with the same macro `FAILED` + `subStatus` `"Entrega fallida"` were skipped by the projection watermark when only `motivo` / `motivoDesc` / `subMotivoDesc` changed. API persisted every `package_events` row; projector showed a single failed attempt.

**Emission rule (closed):** emit on OOLL delivery failure when (1) state delta vs watermark, **or** (2) **failure-context fingerprint** delta (`reasonCode` + `reasonLabel` + `comment`, normalized; Andreani `*` omitted).

**Andreani v1 wire → `deliveryFailureContext` (passthrough — no semantic slug):**

| Raw field | Projected field |
|-----------|-----------------|
| `motivo` | `reasonCode` (e.g. `"50"`) |
| `motivoDesc` (if not `*`) | `reasonLabel` |
| `subMotivoDesc` (if not `*`) | `comment` (courier / sub-reason detail) |

`subMotivo` is used only for status-mapping lookup keys (`GestionTelefonica,50,59`); it is **not** copied to the projector payload.

**API (implemented):** schema fields `lastProjectedFailureReasonCode/Label/Comment`; `PackageProjectionDeltaService.hasDeliveryFailureContextDelta`; watermark upsert **update** path persists failure fields on re-claim (`packageProjectionWatermark.repository.ts`).

**Projector:** no contract change — append timeline by `externalEventId`; `resolveCurrentMilestone` unchanged (B.8).

**Orchestrator (implemented — cases may evolve):** fourth bucket `failed_then_delivered`: `GestionTelefonica,50,57` then `50,59` + `subMotivoDesc` comment → `entregada`; SQL assert ≥2 FAILED timeline rows. See `testing-orchestrator/README.md` bucket D.

**Projector seeder (implemented):** `deliveryHistorySeed.data.ts` — lifecycle stage `multiFailureBeforeDelivered` (2× failed context then `DELIVERED`); unit tests in `deliveryHistorySeed.data.spec.ts`.

**Acceptance B.5.9:** unit delta + watermark claim tests ✅; orchestrator bucket D asserts ✅; deferred: normalized semantic `reasonCode` catalog (out of v1 scope).

**References:** `domain-model.md` §4.2 · §4.4 · §5.1 · `blocks-api.md` §5.4 · `andreaniEvents.md`.

**B.5.10 Andreani E2E scenario coverage (implemented — orchestrator):**

**Problem (closed for orchestrator):** `testing-orchestrator` previously exercised **`GestionTelefonica`** shapes only. Production traffic (see `andreaniEvents.md`) uses **`Visita`**, **`EnvioEntregado`**, and unmapped operational noise.

**Implemented (`testing-orchestrator/lib/andreani/helpers.js`):** five global buckets (sorted `packageExternalId`):

| Bucket | Prod-shaped events | Outcome |
|--------|-------------------|---------|
| A `ooll_reception` | `EnvioConsolidado` | OOLL reception |
| B `delivered` | reception → `Distribucion` → **`EnvioEntregado`** (motivo 99) | `DELIVERED` |
| C `macro_failed` | reception → `Distribucion` → **`Visita`** (26,22) | macro `FAILED` |
| D `failed_then_delivered` | **`GestionTelefonica`** 2× failure → delivered (B.5.9) | `DELIVERED` + ≥2 FAILED timeline |
| E `unmapped_noise` | **`ExpedicionHojaDeRutaDeViaje`** | inbox `PROCESSED`; no `package_events`; projector stays `LAST_MILE_DISPATCHED` |

Steps `05-andreani` / `07-e2e-final-assert` assert bucket E SQL gates. HTTP samples cover all five buckets (soft).

**Seeders aligned:** `deliveryHistorySeed.data.ts` — macro FAILED uses **`Visita`** (26,22); simple DELIVERED uses **`EnvioEntregado`**; multi-failure → delivered retains **`GestionTelefonica`** (B.5.9). API `simulateAndreaniEvents.js` exposes the same scenario names. Unmapped noise (`ExpedicionHojaDeRutaDeViaje`) is manual/API script only — not seeded into projector views (no projection). **B.5.11** (`Visita` + text `"Entregado"`) intentionally excluded.

**Acceptance B.5.10:** ✅ orchestrator + seeders — ≥1 `Visita` coded failure, ≥1 `EnvioEntregado` delivery, existing `GestionTelefonica` bucket D, one unmapped `PROCESSED`-only event (orchestrator). User validation: clean DB → `npm run e2e`.

**B.5.11 Analysis debt — `Visita` + text-only `"Entregado"` (OPEN):**

Production sends `evento: Visita` with `motivoDesc`/`subMotivoDesc: "Entregado"` and **empty** `motivo`/`subMotivo`. Lookup tries key `Visita` only — **not in catalog** → inbox `PROCESSED`, **no** timeline/projection, despite semantic delivery.

**Before mapping or adapter change:** trace how legacy TMS / Delivery Manager consumed these events and what customer-facing state resulted. Document outcome in `plan.md` decision log or `andreaniEvents.md`. See **§ Pending B.5.11**.

**B.8 Última milla — macro, proyección y terminales (`domain-model.md` §2.1, §4.3):**

**Terminales v1 (cierre ciclo):** solo **`DELIVERED`**. **`CANCELLED`** es terminal de dominio pero flujo manual — **fuera de scope activo** (B.8.5 no priorizado).

**`FAILED` no es terminal:** fallos OOLL **siempre proyectan** (macro `FAILED`, `subStatus`, `deliveryFailureContext`, timeline). **No** congelan ciclo — grilla mantiene último progreso. Reintentos → `DELIVERED` permitido.

**Catálogo 5 `MilestoneKey`:** códigos UI/filtro incluyen `FAILED`/`CANCELLED`; solo `DELIVERED` cierra ciclo operativo v1.

| Subtask | Status |
|---------|--------|
| B.8.1–B.8.3, B.8.6–B.8.7 | **DONE** |
| B.8.4 `isTerminal` en mapping | **DONE** infra — **no** usar `milestoneKey: FAILED` v1 |
| B.8.5 cancel manual | **OUT OF SCOPE** v1 |

**Payload fallo OOLL (proyecta, no terminal):** `stateChange.canonicalStatus: FAILED` + `deliveryFailureContext`; **sin** `milestoneKey`.

**Acceptance C.7 (projector — DONE):** `DELIVERED` cierra grilla; macro `FAILED` no avanza `current_milestone`; `resolveCurrentMilestone` en mapper.

---

### Fase C — Projector read model

**Goal:** `current_milestone`, tramo-aware projection, search.

| ID | Task | Files (indicative) | Status |
|----|------|-------------------|--------|
| C.1 | Migration: `current_milestone` on `delivery_history_views` | schema + migration | **DONE** |
| C.2 | Milestone catalog HTTP + mock (`MOCKS_ENABLED`); tabla `milestones` + seeder | `catalog.controller.ts`, `repositories/catalog/`, `seedAppConfig.ts` | **PARTIAL** — endpoint ✅; persistencia pending |
| C.3 | Projection mapper: set `current_milestone` from `milestoneKey` | `packageNewMilestoneProjection.repository.mapper.ts` | **DONE** |
| C.4 | `PackageEvent.shipmentId` = active trail on each projection | same mapper | NOT STARTED |
| C.5 | Block 11: populate `delivery_history_search_index` in projection tx | projection repository + mapper | **DONE** |
| C.6 | Block 11 lectura + Block 12: `GET /shipments?q=&courier=&deliveryType=&milestone=` | `deliveryHistorySearch` repository + `listShipments` | **DONE** |
| C.7 | Terminales 5 keys: Zod, `resolveCurrentMilestone`, tests | ingestor DTO + mapper | **DONE** — reglas § B.8 |
| R2 | Detalle v2: DTO datos normalizados + mocks regenerados — spec `blocks-projector.md` §8.1 | `getShipmentDetail.dto.ts`, use case, `generateDeliveryHistoryMocks.ts` | **DONE** (validar local: `MOCKS_ENABLED=true` + list → detail) |

**Search — performance (post C.5/C.6, funcionalidad cerrada):**

| ID | Task | Spec | Status |
|----|------|------|--------|
| SP.1 | P1: single prefix query + minimal list `select` without `q` + partial listable indexes | `search-performance-spec.md` §P1 | **CODED** — benchmark gate pending |
| SP.2 | P2: denormalización índice / Block 13 particionado | `search-performance-spec.md` §P2 · Block 13 | **DEFERRED** |

---

### Fase D — Dev tooling & validation

| ID | Task | Status |
|----|------|--------|
| D.1 | `POST /ingestor/journeys/:journeyId/reconcile/chunk` | NOT STARTED |
| D.2 | Local reconciliation loop script | NOT STARTED |
| D.3 | Align `.env.example` (API, projector, fenix-local) | PARTIAL — API prod URL `/api/event` ✅; local docs + bulk env **B.7.4** |
| D.4 | Update tracking docs after each phase | DONE (doc poda 2026-06-23) |
| D.5 | `simulateAndreaniEvents.js` — Fenix consume simulator for `andreani-events` | **DONE** |

---

### Parallel: Fenix E2E (orchestrator)

`testing-orchestrator`: `npm run warehouse` (LPC gates); `npm run e2e` (+ Andreani); `npm run validate` (e2e → search gate). Runbook manual `local-testing-guide.md` (EP-ready → WMI → …) still **pending** for full B.6. See `plan.md` § Validation evidence for run artifact rules.

---

## Acceptance Criteria For V1

- Tramos: `FIRST_MILE` ends and `LAST_MILE` starts at `oeLaunchAt`; warehouse ops under `LAST_MILE`.
- Terminales: solo `DELIVERED` cierra ciclo; `FAILED` proyecta sin congelar; `CANCELLED` manual fuera scope v1.
- API → projector E2E through `fenix-local` with idempotent replay.
- OOLL reintentos: cada intento fallido con motivo/comentario distinto proyecta fila timeline (`deliveryFailureContext`); N×`FAILED` → `DELIVERED` auditable (**B.5.9**).
- `current_milestone` + milestone catalog HTTP (5 values) in projector; search index transactional.
- List search: param `q` en `GET /delivery-history/shipments` (sin endpoint aparte); `total` paginado sobre conjunto filtrado; `q` mín. 3 chars (BE 400 / FE no envía antes).
- FE backoffice: debounce 300 ms en `q`; `onFiltersChange` → refetch server-side (ver `blocks-projector.md` Block 11 § contrato frontend).
- `MOCKS_ENABLED` in `app_config` toggles list/detail/milestones mock responses.
- Detalle v2 (`Block R2`): shape estable; datos semánticos (`logistics`, `shipmentInfo`); comercial `null` sin fuente; sin `quickActions` ni presentación UI en API.
- Tests: tramo boundaries, gate payloads, dedupe, projection tx.

---

## Do Not Do

- Do not modify `framework/` or per-project `AGENTS.md` — **excepción:** capa Fenix queue/outbox para B.7 (ya aplicada en B.7.1).
- Do not project `DRAFT_BASE`.
- Do not emit `PACKAGE_NEW_MILESTONE` from WMI sync or journey chunk.
- Do not emit on WMS classification (persist only).
- Do not create `LAST_MILE` shipment in Block 3 chunk.
- Do not treat 11 `*At` fields as 11 UI milestones.
- Do not insert `outbox_ingestor` or upsert watermark from gate use cases (outbox se activa post-return vía `EventPublisherInterceptor` → `FenixOutboxProducer`).
- Do not update projection watermark post-`SENT` (claim en producer tx con outbox `PENDING`; ver `domain-model.md` §5.1).

---

## Decision log (consolidated from design specs)

| ID | Decision | Rationale |
|----|----------|-----------|
| D-01 | **Single status source:** `plan.md` only; `blocks-*.md` are design specs without inline DONE/PENDING | Reduce cognitive load; avoid drift |
| D-02 | **Andreani `deliveryFailureContext` v1:** passthrough wire fields — `motivo`→`reasonCode`, `motivoDesc`→`reasonLabel`, `subMotivoDesc`→`comment`; sentinel `*` omitted | Matches `andreani-webhook` + `andreaniOoll.adapter.ts`; no semantic slug catalog in v1 |
| D-03 | **B.5.9 failure fingerprint:** normalized `reasonCode` + `reasonLabel` + `comment` on watermark; emit on context-only delta | Multi-attempt failed delivery timeline before `DELIVERED` |
| D-04 | **Semantic `reasonCode` (e.g. `DESTINATARIO_AUSENTE`):** deferred post-v1 | Would require catalog + adapter + watermark migration |
| D-05 | **SP.1 search perf:** P1.1–P1.3 implemented; acceptance = Artillery gate @ 250k archived under `testing-orchestrator/runs/` | Iterable until v1 benchmark recorded |
| D-06 | **E2E/search evidence:** reproducible runs, not copied session IDs | See § Validation evidence below |
| D-07 | **Projector seeder:** `multiFailureBeforeDelivered` stage in `deliveryHistorySeed.data.ts` mirrors B.5.9 timeline shape | Mocks + search prefixes stay aligned with seed data |
| D-08 | **`fechaHora`:** business event time at Andreani origin; suffix `Z` = UTC; Redpanda timestamp = ingest latency | API uses `new Date(fechaHora)` as `occurredAt` |
| D-09 | **`idCliente`:** optional on wire (often empty in prod); persisted in inbox raw payload; **not used** for MATCH or projection | Aligns with `andreani-webhook`; MATCH by `idAndreani` only |
| D-10 | **Status lookup keys:** only `evento`, `motivo`, `subMotivo` — **not** `motivoDesc`/`subMotivoDesc` | Desc fields → `deliveryFailureContext` only |
| D-11 | **Unmapped Andreani:** inbox `PROCESSED`, no `package_events`, no projector emission (v1) | e.g. `ExpedicionHojaDeRutaDeViaje` |
| D-12 | **E2E Andreani:** orchestrator + projector seeders aligned on prod shapes (**B.5.10** ✅); B.5.11 Visita+text Entregado still unmapped | Orchestrator E2E + seed timeline fixtures |
| D-13 | **`Visita` + text `"Entregado"` without `motivo` 99:** unmapped today — **analysis before fix** (**B.5.11**) | Trace legacy systems first |

---

## Validation evidence (agents & developers)

**Where artifacts live:** `testing-orchestrator/runs/<timestamp>/` (`report.json`, Artillery summaries).

**Rules:**

1. **Do not cite** run IDs or timestamps from another developer's machine as proof of acceptance.
2. **If no fresh run exists** after material changes (watermark, Andreani buckets, search repo, migrations), **warn the user** before claiming E2E or search perf acceptance.
3. **After relevant changes,** re-run `npm run e2e` and/or `npm run search-load` (or `npm run validate`), archive output, and compare p50/p95 to the previous baseline on the **same** hardware and seed count.
4. **Orchestrator DB:** `PROJECTOR_DATABASE_URL` must match projector `POSTGRES_DATABASE_URL` (see `testing-orchestrator/README.md`).

---

## Fenix Local Integration

**Emulator:** `fenix-local` mirrors **queue-event-generator** (not legacy `/publish`).

| Endpoint | Role |
|----------|------|
| `POST /api/event` | Single message — maps to `FENIX_GENERATOR_URL` |
| `POST /api/event/bulk` | Cohort LPC / dispatch — **B.7.2** outbox chunks secuenciales |
| `POST /publish` | Legacy alias only |

**Ports:** fenix-local `9000`, API `9100`, projector `9200`.

**API local env:**

```bash
FENIX_GENERATOR_URL=http://localhost:9000/api/event
CORE_SKIP_FENIX_QUEUE=false
```

**Subscribers:** see `fenix-local/.env.example`. Key routes:

- `FENIX_JOURNEY_RECONCILIATION_SUBSCRIBERS` → API consume
- `FENIX_PACKAGE_NEW_MILESTONE_SUBSCRIBERS` → projector consume

**Bulk relay design:** § B.7 arriba · **Runbook:** `local-testing-guide.md` §1 · **Emulator:** `fenix-local/README.md`
