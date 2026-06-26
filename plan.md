# Plan.md - Current V1 Completion Plan

Operational plan for `shipments-history-api/`, `shipments-history-projector/`, `fenix-local`.

**Única fuente de status e implementación.** Dominio: `domain-model.md`. Onboarding: `prompt.md`.

## Doc map

| Doc | Rol |
|-----|-----|
| **`plan.md`** | Status, checklist, orden B.x/C.x/D.x |
| `domain-model.md` | Milestones, tramos, gates, watermark (sin status) |
| `prompt.md` | Onboarding corto + links |
| `local-testing-guide.md` | Runbook E2E §1 |
| `blocks-api.md` / `blocks-projector.md` | Spec diseño por block (status desactualizado → ignorar); **search funcional → `plan.md` C.5–C.6**; **search perf → `search-performance-spec.md`** |
| `backoffice-list-search-integration.md` | Contrato integración FE (list `q` + filtros) |
| `search-performance-spec.md` | Optimizaciones pendientes de lectura search (P1/P2) |

**Last updated (2026-06-25):** **C.5 + C.6 search** ✅ (índice en proyección + `GET /shipments?q` + filtros). **B.7** bulk LPC ✅; **B.5.8** `OollProcessingService` ✅. **Next projector:** search **performance** P1 (`search-performance-spec.md`) · **C.4** tramo en `PackageEvent.shipmentId` · **C.2** persistencia catálogo `milestones`. **Next API:** B.7.4 docs → B.8.5 → B.6 runbook manual.

**Diseño operativo cross-cutting:** terminales `FAILED`/`CANCELLED` → **§ B.8** · Fenix bulk LPC → **§ B.7**

---

## Current Assessment

Warehouse path through dispatch gate is **coded on API** (tramos + gate payloads + WMS 6.4/6.5). Remaining gaps before production E2E:

| Gap | Status |
|-----|--------|
| Projection watermark (read gate + claim in producer tx) | **B.2 — DONE** (B.2.3 gates + B.2.4 producer claim ✅; B.2.5–B.2.6 retry/idempotency tests pending) |
| Block 5 Andreani / OOLL incremental | **B.5 — DONE (MVP v1)** — filtro watermark ✅ **B.2.3**; replay UNMATCHED v2 deferred |
| OOLL shared engine (`OollProcessingService`) | **B.5.8 — DONE** (`blocks-api.md` §5.9; E2E `2026-06-25T16-33-08-710Z`) |
| API → projector E2E via `fenix-local` (warehouse LPC + optional OOLL) | **B.6 — PARTIAL** — orchestrator `warehouse` ✅ `2026-06-25T16-59-55-015Z`; `e2e` ✅ `2026-06-25T16-33-08-710Z`; runbook manual EP-ready → WMI pending |
| Fenix bulk relay (`/api/event/bulk`) in API producer | **B.7 — DONE (core)** — B.7.0–B.7.3 ✅; B.7.4 docs pending; warehouse E2E `2026-06-25T16-59-55-015Z` |
| **`FAILED` / `CANCELLED` milestones (API emission)** | **B.8 — PARTIAL** (B.8.1–B.8.4, B.8.6–B.8.7 ✅; B.8.5 manual cancel pending) |
| Projector `current_milestone` + `milestoneKey` handling | **Fase C — PARTIAL** (C.1 + C.3 + **C.7** ✅; **C.5 + C.6** ✅; C.4 pending) |
| Search index + list `q`/filtros | **Fase C — DONE (C.5 + C.6)** |
| Search lectura — performance P1 | **PENDING** — `search-performance-spec.md` (funcionalidad cerrada) |
| Detalle HTTP v2 (`GetShipmentDetailOutputDto`) | **Block R2 — DONE** (spec §8.1 ✅; mocks regenerados; E2E manual con `MOCKS_ENABLED=true`) |

---

## Ground Truth By Codebase

| Area | Status | Evidence | Notes |
|---|---|---|---|
| API Block 2 ep-ready | **DONE** | `consumeEpReadyToCollectOrders.useCase.ts`, `repositories/shipment/` | `FIRST_MILE` shipment + link on ep-ready |
| API Block 3 WMI + LPC | **DONE** | `processJourneyReceiptChunk.useCase.ts` | `LAST_MILE` identifier only; no premature `Shipment LAST_MILE` |
| API Block 4 outbox | **DONE (sync)** | `fenix.outbox.producer.ts`, `eventPublisher.interceptor.ts` | Outbox+watermark bulk tx ✅; Fenix LPC relay via **`produceBulk`** chunks ✅ **B.7.2** |
| API Block 6.3 OE_LAUNCH | **DONE** | `consumeOeLaunch.useCase.ts`, `oeLaunch.repository.ts`, `shipment.repository.ts` | Tramo transitions + `WAREHOUSE_RECEIVED` payload |
| API Block 6.4 classification | **DONE** | `consumeWmsPackageClassification.useCase.ts` | Persist only; handler `ASSIGN_ITEM_TO_CONTAINER` |
| API Block 6.5 dispatch | **DONE** | `consumeWmsContainerPackagesShipped.useCase.ts`, `wmsDispatch.repository.ts` | Bulk SQL; gate `LAST_MILE_DISPATCHED` on shipped |
| API watermark | **DONE (B.2.4)** | `PackageProjectionOutboxCoordinator`, `FenixOutboxProducer` tx hook | B.2.5–B.2.6 retry tests |
| API Block 5 Andreani | **DONE (MVP v1)** | `consumeAndreaniEvents.useCase.ts`, `repositories/andreani/`, `application/oollProcessing/` | B.5.1–B.5.8 ✅ |
| Projector bootstrap consumer | **DONE (C.3 minimal)** | `applyPackageNewMilestone.useCase.ts` | Sets `current_milestone` from `milestoneKey` (forward-only pipeline) |
| Projector `current_milestone` | **DONE (C.1 + C.3 + C.7)** | migration + mapper terminales | C.2 catalog HTTP ✅ (static); tabla `milestones` pending |
| Search index + list `q`/filtros | **DONE (C.5 + C.6)** | `packageNewMilestoneProjection` + `deliveryHistorySearch` repository | Mocks decorator + Artillery load ✅; perf P1 → `search-performance-spec.md` |
| Projector milestone catalog | **PARTIAL (C.2)** | `catalog.controller.ts`, `repositories/catalog/milestoneCatalog.data.ts` | HTTP + mock; persistencia pending |
| API → Projector E2E | **PARTIAL (B.6)** | `testing-orchestrator` | `warehouse` ✅ `2026-06-25T16-59-55-015Z`; `e2e` ✅ `2026-06-25T16-33-08-710Z`; runbook manual pending |
| `fenix-local` | **DONE (generator emulator)** | `fenix-local/index.js`, `lib/gateway.js` | `/api/event`, `/api/event/bulk`; legacy `/publish` |

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
| B.5 | Block 5 Andreani: MATCH → incremental OOLL + `DELIVERED`; UNMATCHED persistido (forward-only, replay v2) | `consumeAndreaniEvents.useCase.ts`, `repositories/andreani/`, `oollProcessing/` | **DONE (MVP v1)** — B.5.8 ✅ |
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
| B.5.1 | Extender `EventStatus` con `UNMATCHED` + índice parcial inbox Andreani | `schema.prisma`, `20260623120000_add_inbox_andreani_unmatched_index` | **DONE** |
| B.5.2 | Lookup package por `idAndreani` (criterio MATCH §5.8.2) | `andreani.repository.ts` → `findMatchByTrackingId` | **DONE** |
| B.5.3 | Integrar `CarrierStatusMapping` + matriz §5.3 | `oollProcessing.service.ts` (+ Andreani facade) | **DONE** |
| B.5.4 | MATCH: persist milestones/events + outbox incremental / gate `DELIVERED` | `andreani.repository.ts` + use case | **DONE** |
| B.5.5 | NO MATCH: inbox `UNMATCHED`, sin dominio/outbox (forward-only) | `consumeAndreaniEvents.useCase.ts` | **DONE** |
| B.5.6 | Tests: MATCH, UNMATCHED, idempotencia, outbox, sin replay v1 | `test/unit/ingestor/consumeAndreaniEvents.useCase.spec.ts` | **PARTIAL** — 6 tests; falta `ambiguous_match` explícito |
| B.5.7 | Replay UNMATCHED | — | **v2 — fuera MVP** |
| B.5.8 | Extract shared `OollProcessingService` (matrix + watermark + outbox); thin per-carrier consumers | **`blocks-api.md` §5.9** | **DONE** — E2E `2026-06-25T16-33-08-710Z` |

**B.5.8 scope (summary — detail in `blocks-api.md` §5.9 only):**

- Move Block 5.3 matrix, watermark delta, listable gate, and `PACKAGE_NEW_MILESTONE` assembly out of `consumeAndreaniEvents.useCase.ts` into `application/oollProcessing/`.
- Andreani keeps: DTO, Fenix handler, MATCH/UNMATCHED, `repositories/andreani/`, raw-key adapter.
- Future operators (OCASA, ClicPaq, MOOVA, …): new consumer + repository + adapter; **reuse** `OollProcessingService`.
- **Mandatory:** rewrite/split unit tests (`oollProcessing.service.spec.ts` + thinned Andreani spec). No behavior change for Andreani (orchestrator E2E must pass unchanged).
- **CRITICAL:** implementing agents **must not run terminal commands** — human/CI runs build, unit tests, and E2E.

**B.8 Terminal milestones `FAILED` / `CANCELLED` (single source — reglas de dominio en `domain-model.md` §2.1, §4.1, §4.3):**

**Scope:** API **B.8** (salvo B.8.5 manual) ✅ · Projector **C.7** ✅ · Catálogo HTTP **C.2** partial.

**Catálogo `MilestoneKey` v1 (5 valores):**

| `code` | Label UI (es) | Tipo | Trigger principal |
|--------|---------------|------|-------------------|
| `WAREHOUSE_RECEIVED` | Recibido en depósito | Progreso | `OE_LAUNCH` (MVP: **gate listable**) |
| `LAST_MILE_DISPATCHED` | Despachado a última milla | Progreso | `CONTAINER_PACKAGES_SHIPPED` |
| `DELIVERED` | Entregado | Terminal éxito | OOLL entrega confirmada |
| `FAILED` | Entrega fallida | Terminal fallo **o** macro transitorio | OOLL entrega fallida (ver §FAILED) |
| `CANCELLED` | Cancelado | Terminal absoluto | Manual declarativo; Andreani `Anulada` |

`DELIVERED`, `FAILED` y `CANCELLED` son **macro y milestone** (`canonicalStatus` = mismo valor en `stateChange`).

**`canonicalStatus` v1:** `IN_TRANSIT` (ciclo normal) · `DELIVERED` · `FAILED` · `CANCELLED` · `CREATED` (draft API — no proyectable).

**Listabilidad (`isListable` / grilla):** package **no aparece en grilla** hasta cruzar el **primer gate listable**. MVP warehouse = `WAREHOUSE_RECEIVED` (`OE_LAUNCH`).

| Escenario | Grilla | Emisión projector |
|-----------|--------|-------------------|
| Cancel / `Anulada` **antes** del gate listable | **No** | Persist API; **no** emitir `PACKAGE_NEW_MILESTONE` (MVP) |
| Cancel / `Anulada` **después** del gate listable | **Sí** (`current_milestone = CANCELLED`) | `milestoneKey: CANCELLED` |
| Futuro sin warehouse | Otro entry gate | Misma regla genérica |

**Máquina de estados — terminales:**

```text
Progreso (orden estricto): WAREHOUSE_RECEIVED → LAST_MILE_DISPATCHED
Terminales: DELIVERED | FAILED | CANCELLED
```

| Transición | ¿Permitida? |
|------------|-------------|
| Progreso → terminal (post-gate listable) | Sí |
| `FAILED` → `DELIVERED` | **Sí** — reintentos OOLL |
| `DELIVERED` → `FAILED` | **No** |
| `*` → `CANCELLED` (post-gate listable) | **Sí** |
| `CANCELLED` → `*` | **No** — terminal absoluto |

**Projector `current_milestone`:** no pipeline lineal único — `resolveCurrentMilestone` en mapper (**C.7** ✅).

**`FAILED` — dos niveles (tolerancia MVP):**

| Situación | `canonicalStatus` | `milestoneKey` | Grilla |
|-----------|-------------------|----------------|--------|
| Entrega fallida **reintentable** (mayoría OOLL) | `FAILED` | `null` | Sin cambio — último progreso |
| Entrega fallida **terminal** (fase 2: mapeos carrier) | `FAILED` | `FAILED` | `FAILED` |
| Entrega exitosa posterior | `DELIVERED` | `DELIVERED` | `DELIVERED` |

Fase 1 API ✅: macro `FAILED` reintentable sin `milestoneKey`. Fase 2: `CarrierStatusMapping` flag terminal → `milestoneKey: FAILED` (infra ✅ B.8.4; keys negocio pending).

**Ejemplos payload `PACKAGE_NEW_MILESTONE` (fragmentos `data` — sin wrapper Fenix):**

Reintentable — macro `FAILED`, **sin** `milestoneKey` (no avanza grilla):

```json
{
  "stateChange": {
    "status": "FAILED",
    "canonicalStatus": "FAILED",
    "subStatus": "Entrega fallida",
    "lastOccurredAt": "2026-06-13T16:45:00.000Z"
  },
  "deliveryFailureContext": {
    "reasonCode": "DESTINATARIO_AUSENTE",
    "reasonLabel": "Destinatario ausente",
    "comment": "Timbre sin respuesta"
  }
}
```

Terminal — `milestoneKey: FAILED` (fase 2 / mapeo carrier terminal):

```json
{
  "milestoneKey": "FAILED",
  "stateChange": {
    "status": "FAILED",
    "canonicalStatus": "FAILED",
    "subStatus": "Entrega fallida",
    "lastOccurredAt": "2026-06-13T16:45:00.000Z"
  },
  "deliveryFailureContext": {
    "reasonCode": "DESTINATARIO_AUSENTE",
    "reasonLabel": "Destinatario ausente",
    "comment": "Timbre sin respuesta"
  }
}
```

**`CANCELLED`:**

| Origen | v1 |
|--------|-----|
| Andreani `Anulada` | Automático → `CANCELLED`; emite solo post-gate listable ✅ **B.8.3** |
| Manual / declarativo | **B.8.5** — gate nuevo (use case + contrato Fenix) |
| `Suspendida`, `En devolución`, etc. | Persist API; emisión diferida |

Post-gate listable — ejemplo payload:

```json
{
  "milestoneKey": "CANCELLED",
  "stateChange": {
    "status": "CANCELLED",
    "canonicalStatus": "CANCELLED",
    "subStatus": "Anulada",
    "lastOccurredAt": "2026-06-13T10:00:00.000Z"
  },
  "metadata": {
    "emissionTrigger": "OOLL_CANCELLED",
    "andreaniTrackingId": "3639161161",
    "rawEvento": "Anulada"
  }
}
```

(`metadata.emissionTrigger`: `OOLL_CANCELLED` para Andreani `Anulada`; `MANUAL_CANCEL` cuando exista **B.8.5**.)

`cancellationContext` (motivo, actor, ticket): opcional v1 — no bloquea B.8.

| Subtask | Qué | Dónde | Status |
|---------|-----|-------|--------|
| B.8.1 | Enum `ProjectionMilestoneKey` + `FAILED`, `CANCELLED` | `schema.prisma`, migration | **DONE** |
| B.8.2 | OOLL reintentable: macro `FAILED`, sin `milestoneKey` | `consumeAndreaniEvents.useCase.ts` | **DONE** |
| B.8.3 | Andreani `Anulada` → `CANCELLED`; emisión post-gate listable | mismo use case | **DONE** |
| B.8.4 | Flag terminal en `CarrierStatusMapping` → `milestoneKey: FAILED` | schema, entity, use case | **DONE** (infra; marcar keys negocio pending) |
| B.8.5 | Gate manual cancel declarativo | TBD use case / consumer | NOT STARTED |
| B.8.6 | Helper package cruzó gate listable | `packageListableGate.service.ts` | **DONE** |
| B.8.7 | Tests FAILED macro, Anulada, DELIVERED sobre FAILED | `consumeAndreaniEvents.useCase.spec.ts` | **DONE** |

**Acceptance B.8 (API):** enum `FAILED`/`CANCELLED`; OOLL reintentable sin `milestoneKey`; `Anulada` post-gate emite `CANCELLED`, pre-gate solo persist; watermark permite `DELIVERED` tras macro `FAILED`; (fase 2) mapeos terminal → `milestoneKey: FAILED`.

**Acceptance C.7 (projector — DONE):** grilla muestra `FAILED`/`CANCELLED` en `current_milestone`; `DELIVERED` reemplaza `FAILED`; `CANCELLED` no regresa; pre-gate cancel no en `listShipments`. Catálogo: `GET /delivery-history/catalog/milestones` (seed estático + mock `MOCKS_ENABLED`); tabla `milestones` pending **C.2**.

**No tocar API para C.7/C.2 projector** — contrato estable post-B.8.

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
| SP.1 | P1: query única prefix + `select` mínimo list sin `q` + índices parciales listables | `search-performance-spec.md` §P1 | **PENDING** |
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

`testing-orchestrator`: `npm run warehouse` (LPC gates) ✅ `2026-06-25T16-59-55-015Z`; `npm run e2e` (+ Andreani) ✅ `2026-06-25T16-33-08-710Z`. Runbook manual `local-testing-guide.md` (EP-ready → WMI → …) still **pending** for full B.6.

---

## Acceptance Criteria For V1

- Tramos: `FIRST_MILE` ends and `LAST_MILE` starts at `oeLaunchAt`; warehouse ops under `LAST_MILE`.
- Gates: **5** `MilestoneKey` values; list UI uses `current_milestone`; OOLL reintentable macro `FAILED` without milestone advance.
- Terminales: `DELIVERED` over `FAILED` allowed; `CANCELLED` absolute; pre-gate cancel not listable.
- API → projector E2E through `fenix-local` with idempotent replay.
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
