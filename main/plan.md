# plan.md — Delivery History (single source of truth)

Operational + design source for `shipments-history-api/`, `shipments-history-projector/`, `fenix-local/`.

> **Read only this file.** It carries current status **and** the full project context (rules, architecture, domain summary, design decisions, testing). Deep references when you actually need them:
> - **Domain authority:** `domain-model.md` (milestones, tramos, gates, watermark). If it conflicts with this file on a domain definition, `domain-model.md` wins.
> - **Business / physical context:** `logistics.md` (what happens physically: EnvíoPack, WMS, OOLL, real wire shapes).
> - **Andreani production wire samples:** `andreaniEvents.md`.
> - **E2E, local runbook, SQL validation, search load:** `testing-orchestrator/README.md`.
> - **Implemented design lives in code + tests** — do not re-derive it from archived specs.

> **Before writing code in a project, load that project's `AGENTS.md`** (`shipments-history-api/AGENTS.md` or `shipments-history-projector/AGENTS.md`): hexagonal repository + mapper + provider pattern, strict TypeScript, Fenix consumer pattern. `framework/` is frozen — never modify.

---

## 1. Current status

**Warehouse MVP v1 is coded and functionally coherent** across API, Fenix relay, and projector. The happy path is complete: tramo boundaries (`FIRST_MILE`/`LAST_MILE`), `OE_LAUNCH` and dispatch gates, projection watermark claim, bulk Fenix relay, projector list/search/detail, `current_milestone` rules, and Andreani MATCH / UNMATCHED.

**Not yet closed for v1:** `C.4` (projected shipment link + `segments[]` are not active-trail complete) and `C.2` (milestone catalog persistence decision). `B.5.11` is **analysis debt** (not a code task). `SP.1` needs a fresh Artillery gate. `D.1`/`D.2` only if non-LPC reconciliation tooling stays in v1 scope.

### Pendiente (v1 — foco implementación)

| ID | Qué | Dónde | Estado |
|----|-----|-------|--------|
| **C.4** | `PackageEvent.shipmentId` = tramo activo; **`segments[]`** hoy ≈1 (FIRST_MILE no proyecta). Mapper recibe un único `shipmentId`; falta trail activo | projector mapper/repo | **NOT STARTED** |
| **C.2** | Decidir: tabla `milestones` + seeder + lectura DB en `listActiveMilestones()`, **o** descopar formalmente a estático (HTTP catálogo ✅ static hoy) | projector | **PARTIAL** |
| **SP.1** | Search read perf — P1.1–P1.3 coded; **re-run Artillery gate @ 250k** y archivar summary (`testing-orchestrator/runs/`) | projector | **IN PROGRESS** (user-run) |
| **B.7.4 / D.3** | `.env.example` a defaults locales (ports `9100/9200/9000`, `FENIX_GENERATOR_URL=localhost:9000`) + docs bulk Fenix | API, projector, `fenix-local/` | **PARTIAL** — hoy apuntan a generator remoto + `APP_PORT=5000` |
| **B.6** | Fresh reproducible E2E evidence on a clean DB (runbook ya documentado en `testing-orchestrator/README.md`) | testing-orchestrator | **PENDING evidence** |
| **B.5.11** | **Analysis (debt), no code fix:** `Visita` + text-only `"Entregado"` (sin `motivo` 99). Trazar interpretación legacy TMS/Delivery Manager **antes** de tocar mapping; registrar en Decision log | ops + API | **OPEN** |
| **B.2.6** | Tests: idempotent-claim watermark + replay depth | API | **PARTIAL** |
| **D.1** | `POST /ingestor/journeys/:journeyId/reconcile/chunk` (solo si non-LPC reconcile sigue en scope v1) | API | **NOT STARTED** |
| **D.2** | Script reconciliation loop local | tooling | **NOT STARTED** |
| **EP.1** | Identidad de drafts `EP_READY` con `EP_BASE` reutilizado (key sintética `EP_BASE + receivedDateEpochMs`, columna `ep_base` searchable no-unique, promoción `readyToCollectAt` del draft `OPEN` más nuevo, lifecycle draft). Spec completa: `duplicated-ep-tracking.md` | API + seeders + orchestrator | **SPEC** (not started) |

**Fuera de scope v1 activo:** cancel manual (`B.8.5`); `milestoneKey: FAILED` como terminal (fallos OOLL **proyectan** macro `FAILED`, no congelan ciclo); SP.2 / Block 13 (particionado + TTL search); UNMATCHED replay v2; inbox/outbox monthly partitioning + closure tables (forward-looking — see § 3.1).

### Hecho (warehouse MVP v1)

API: tramos, gates, watermark, bulk LPC, OOLL Andreani (`B.5.8` shared engine, `B.5.9` failure-context delta, `B.5.10` prod-shaped buckets). Projector: proyección, `current_milestone`, search (Block 11/12), detalle R2. E2E orchestrator + Artillery. Detalle por fase en § 6 (tablas A–D).

---

## 2. Agent rules & runtime

1. Do **not** modify `framework/` (frozen in both projects). Exception applied: Fenix queue/outbox layer for B.7.
2. Repository + mapper + provider pattern; strict TypeScript (no `any`). See per-project `AGENTS.md`.
3. **Never** edit generated mocks by hand — regenerate with project scripts.
4. Tests are phase 2 unless explicitly requested (TDD).

**Runtime split (critical):**

| Acción | Agente | Usuario |
|--------|--------|---------|
| `start:dev`, Docker, build, test, lint, orchestrator, Artillery | **Never** | Yes |
| Edit code, read docs | Yes | — |
| SQL read-only (MCP `shipments-history-postgres`) | Yes, scoped | — |

**MCP Postgres:** multidb `localhost:5432` — `shipments-history-api` | `shipments-history-projector`. One DB per turn; no cross-DB. Queries with `LIMIT` / by `id` (large dataset).

**Local ports:** fenix-local `9000`, API `9100`, projector `9200`. Flow: `API → outbox → fenix-local → projector (PACKAGE_NEW_MILESTONE)`.

**Authority order:** `domain-model.md` (domain) → `plan.md` (status + design) → code (implementation). FE contracts (list search, detail) were handed to teams; they do not live in this repo.

---

## 3. Architecture & system map

Two services with non-overlapping responsibilities, plus a local Fenix emulator:

- **`shipments-history-api` (write-heavy):** consumes Fenix queues, validates idempotency, persists the event store, runs tramo/gate logic, emits `PACKAGE_NEW_MILESTONE` via outbox. HTTP entry: `POST /ingestor/fenix-queue/consume`.
- **`shipments-history-projector` (read-heavy):** consumes `PACKAGE_NEW_MILESTONE`, builds the listable read model (`delivery_history_views`), search index, detail; exposes list/detail/catalog APIs.
- **`fenix-local`:** mirrors the **queue-event-generator** (`POST /api/event`, `POST /api/event/bulk`); `POST /publish` is a legacy alias only.

### 3.1 Persistence model (package-centric)

**Trazabilidad por bulto primero; reconciliación con entidades logística/comercial después.** The `Package` is the canonical traceability root; events are append-only against it regardless of leg.

| Entity | Role |
|--------|------|
| `Package` | Physical parcel; `externalId` is the cross-source dedupe key (`EP012933329N-01`, last-mile id, …) |
| `PackageEvent` | Append-only timeline; `UNIQUE(packageId, source, externalEventId)`; ordered by `occurredAt` (Event Time, not arrival) |
| `PackageMilestones` | Snapshot of operational `*At` timestamps (layer A) — **not** UI milestones |
| `PackageIdentifierLink` | N:1 map of heterogeneous ids (EP base/full, last-mile, container, entry-order, order) → package; enables out-of-order arrival |
| `Shipment` | A **leg** with an operator (`trailType` `FIRST_MILE`/`LAST_MILE`, `carrierCode`, `trackingNumber`) |
| `PackageShipment` | N:M membership with `enteredAt`/`exitedAt` |
| `Delivery` / `Order` | Commercial context above the leg; `Delivery` can exist before its `Order` (retroactive link by `orderId`) |
| `Journey` | LPC collection trip (WMI operational entity); correlates first mile, does **not** replace `Shipment` |

**Ingest pattern (inbox/outbox):** persist raw first (latency-min), normalize in processing, emit via transactional outbox. Idempotency anchor: `(source/carrier, externalEventId)`. Event Time (`occurredAt` / `fechaHora`) orders timelines; arrival time is pipeline latency. Andreani is a raw-passthrough source: `carrier_status_mappings` translate raw → canonical.

> **Forward-looking (NOT v1):** N shipments per Delivery with **partial delivery as a first-class case**, inbox/outbox monthly RANGE partitioning + auto-partition procedures, closure tables for dynamic order splitting, and worker-claim mechanism choice (optimistic `version` vs `FOR UPDATE SKIP LOCKED`). Build so these can be absorbed without a paradigm change.

> **Draft sub-state (spec `EP.1`, `duplicated-ep-tracking.md`):** `EnvíoPack` reutiliza `EP_BASE` (`trackingNumber`) para órdenes distintas. Para no colapsar anuncios `EP_READY` sobre un mismo `Package`, los drafts tempranos llevan key sintética `EP_BASE + receivedDateEpochMs` y un lifecycle en `metadata.draftState` (`OPEN` → `PROMOTED` / `SUPERSEDED`). El draft **no** proyecta y **no** toma el link `EP_BASE` (reservado al package canónico). La unicidad canónica con `EP_BASE`/`EP_FULL` reutilizado queda **diferida** (se resuelve sólo la capa draft). `Package` sigue siendo la raíz canónica.

### 3.2 Projection watermark (`package_projection_watermarks`)

Ledger of what the API **claimed** to project per package (not the projector read model). Fields: `lastMilestoneKey`, `projectedTimestamps` (JSON), `lastProjectedCanonicalStatus`/`SubStatus`, `lastProjectedFailureReasonCode`/`Label`/`Comment` (B.5.9 fingerprint), `lastProjectedAt`.

Two responsibilities, never mixed:

| Question | Layer | When |
|----------|-------|------|
| Is there a delta / what goes in the payload? | gate use case (+ watermark **read**) | sync, **before** `return { event }` |
| Claim projection + enqueue outbox? | `FenixOutboxProducer` (via interceptor) | **post-return** |
| Did it reach Fenix? | `outbox_ingestor` (`PENDING`→`SENT`/`FAILED`) | producer relay |

Watermark **write** (upsert per package) happens **only** in `FenixOutboxProducer`, in the same tx as `outbox_ingestor PENDING`, before the Fenix relay. Never update post-`SENT`. Replay safety: warehouse `appliedPackageIds` + set-once milestones + `inbox_events.external_event_id` UNIQUE + deterministic `externalEventId` (projector dedup). Full design: `domain-model.md` §5.1.

---

## 4. Domain summary (full authority: `domain-model.md`)

**Four layers (do not confuse):** A) operational timestamps (`*At`), B) timeline `PackageEvent`s, C) macro `canonicalStatus` + `subStatus`, D) projection `MilestoneKey`. The 11 `*At` fields are layer A, **not** 11 UI milestones.

**Macro `canonicalStatus` v1:** `IN_TRANSIT` (default incl. warehouse + non-failed OOLL), `DELIVERED` (terminal success), `FAILED` (retryable OOLL failure — projects, does **not** freeze cycle), `CANCELLED` (manual, out of active scope), `CREATED`/`DRAFT_BASE` (not projectable). `subStatus` is timeline/diagnosis only — never a grid filter, never opens a milestone.

**Tramos (binary v1):** `FIRST_MILE` = ep-ready → physical CD arrival (`oeLaunchAt`); `LAST_MILE` = `oeLaunchAt` → delivered (warehouse ops + OOLL + delivery). No `WAREHOUSE_INTERNAL` third leg. At `OE_LAUNCH`: close first-mile `PackageShipment` (`exitedAt`), open last-mile (`enteredAt`), link last-mile carrier shipment.

**`MilestoneKey` catalog (5) + projection gates:**

| # | Gate / trigger | `MilestoneKey` | Emits | Advances milestone |
|---|----------------|----------------|-------|--------------------|
| — | Block 2 ep-ready / Block 3 chunk / Block 6.4 classification | — | **No** (persist only) | — |
| 1 | `OE_LAUNCH` (`WM2_RS_MAIN_OE_LAUNCH`) | `WAREHOUSE_RECEIVED` | Yes (bootstrap; consolidates first mile) | yes |
| 2 | `CONTAINER_PACKAGES_SHIPPED` | `LAST_MILE_DISPATCHED` | Yes | yes |
| 3 | OOLL checkpoints | — | Yes (mandatory) | no (macro `IN_TRANSIT`) |
| 3b | OOLL failed delivery | — | Yes | no (macro `FAILED` + `deliveryFailureContext`) |
| 4 | OOLL delivered (`EnvioEntregado`/code 99) | `DELIVERED` | Yes | terminal |

`current_milestone` = last applied key per §4.3 rules (forward-only progress; macro `FAILED` does not advance; `DELIVERED` terminal; `DELIVERED→FAILED` forbidden). First listable gate = `WAREHOUSE_RECEIVED` (cancel before it → not in grid).

**Andreani v1 wire mapping** (passthrough → `andreaniOoll.adapter.ts`): `motivo`→`reasonCode`, `motivoDesc`→`reasonLabel` (omit `*`), `subMotivoDesc`→`comment` (omit `*`); `subMotivo` is lookup-only. Status-mapping lookup keys use **`evento`, `motivo`, `subMotivo` only** — never desc fields. Unmapped shapes (e.g. `ExpedicionHojaDeRutaDeViaje`) → inbox `PROCESSED`, no events, no projection.

---

## 5. CRITICAL — non-warehouse scalability

> **v1 is warehouse-first, but the model must NOT assume everything passes through the CD.** EnvíoPack already routes high-volume parcels (heladera, AA split, "bigger/big/maxi") to last-mile OOLL **directly, without the Frávega warehouse**, for space reasons. Those parcels never hit `OE_LAUNCH`, so the current listable entry gate (`WAREHOUSE_RECEIVED`) and the bootstrap-on-`OE_LAUNCH` projection path **do not cover them**.

Design rules to keep this open (`logistics.md`, `domain-model.md` §4.1):

- `Package` stays the single traceability root; OOLL events attach via `PackageIdentifierLink` even with no warehouse milestones.
- A non-warehouse flow needs a **different listable entry gate** than `WAREHOUSE_RECEIVED` (decision pending; do not hardcode warehouse as the only bootstrap).
- The projector must be able to build a view + timeline for a package that has **only** last-mile (OOLL) events.
- Retail (own stock) and full-own-fleet are out of MVP but must be structurally analogous (new `sourceSystem`, new `trailType`, same N:M legs).
- Partial delivery (one parcel of an order delivered, another in transit) must be treatable as a first-class case, not a border case.

This is the main known structural risk for v2; flag it before any change that bakes in "all packages go through the warehouse."

---

## 6. Implementation status by phase

### Fase A — Tramos API

| ID | Task | Status |
|----|------|--------|
| A.1 | Block 2: create `Shipment FIRST_MILE` + `PackageShipment` on ep-ready | **DONE** |
| A.2 | Block 3: chunk keeps `LAST_MILE` identifier link only (no `LAST_MILE` shipment) | **DONE** |
| A.3 | `OE_LAUNCH`: close first mile (`exitedAt`), open `LAST_MILE`, first-mile timestamps in payload | **DONE** |
| A.4 | Unit tests for tramo boundaries | **DONE** |

### Fase B — Gates & emission contract

| ID | Task | Status |
|----|------|--------|
| B.1 | Bootstrap + dispatch gate payloads (`milestoneKey`, `IN_TRANSIT`, `operationalTimestamps`, `trailTransitions`/`warehouseContext`) | **DONE** |
| B.2 | Projection watermark — read in gates + claim in producer | **DONE** (see B.2.x) |
| B.3 | Block 6.4 classification: persist only, no outbox | **DONE** |
| B.4 | Block 6.5 dispatch: gate `LAST_MILE_DISPATCHED` + bulk transactional apply | **DONE** |
| B.5 | Block 5 Andreani: MATCH → incremental OOLL + `DELIVERED`; UNMATCHED persisted (forward-only) | **DONE (MVP v1)** |
| B.6 | E2E via `fenix-local` (EP-ready → WMI → reconcile → OE_LAUNCH → classification → dispatch → projector) | **PARTIAL** — orchestrator `warehouse` + `e2e` ✅; fresh reproducible evidence on clean DB pending (runbook in `testing-orchestrator/README.md`) |
| B.7 | Fenix bulk relay: bulk outbox → sequential chunks → `produceBulk` | **DONE (core)** — B.7.4 docs pending |

**B.2 subtasks:**

| ID | Qué | Status |
|----|-----|--------|
| B.2.1 | Schema `package_projection_watermarks` | **DONE** |
| B.2.2 / B.2.2b | Repo (`findByPackageIds`, `claimBulk`) + delta service | **DONE** |
| B.2.3 | Watermark read + delta filter in gates | **DONE** |
| B.2.4 | Tx claim: upsert watermark + `createManyAndReturn` outbox `PENDING` | **DONE** |
| B.2.5 | Relay retry via existing `eventId` without re-claim | **DONE** — `publishEvent` updates by `eventId` (no new row); `publishManyEvents` routes `eventId` events to the sequential path |
| B.2.6 | Integration tests: gate replay without `event`, producer idempotent claim | **PARTIAL** — gate replay + producer tx ✅; idempotent-claim depth pending |

**B.7 Fenix bulk relay (design + status):** LPC mass gates (`OE_LAUNCH`, `CONTAINER_PACKAGES_SHIPPED`) return `EventResponse[]` (tens–hundreds). Relay uses `POST /api/event/bulk` in chunks of `FENIX_BULK_CHUNK_SIZE` (default 100, = generator `LimitSendMessagesBulk`), **sequential, no `Promise.all` between chunks**. Andreani incremental (1 event) and `eventId` retries stay on sequential `POST /api/event`.

```text
1. tx: watermark claim + createManyAndReturn outbox (PENDING)   ← B.2.4
2. preparedEvents + createdRows in memory (no outbox re-query)
3. for each chunk of N (outbox ids ↔ payloads aligned):
     produceBulk(chunk) → events_with_key[].key = event.identifier
     updateMany outbox: chunk OK → SENT | chunk fail → FAILED (whole chunk)
4. next chunk only after closing the previous one's state
```

HTTP: `200 OK`→`SENT`; `207`/`500`/network → whole chunk `FAILED` (generator does not say which rows). Re-send of a failed chunk is safe (deterministic `externalEventId`, projector dedup). Bulk env vars: `FENIX_GENERATOR_URL[_BULK]`, `FENIX_BULK_ENABLED` (default true), `FENIX_BULK_MIN_SIZE` (2), `FENIX_BULK_CHUNK_SIZE` (100). **Do not:** `Promise.all` or fixed `setTimeout` between chunks; bulk for `eventId` retries; projector bulk consume. Outbox pattern (do not break): gate use case returns `{ event }`; `EventPublisherInterceptor` activates outbox **post-return**; the use case never writes `outbox_ingestor` or watermark.

| ID | B.7 subtask | Status |
|----|-------------|--------|
| B.7.0 | fenix-local `/api/event` + `/api/event/bulk` | **DONE** |
| B.7.1 | `FenixQueueProducer.produceBulk` + env + 207/500 errors | **DONE** |
| B.7.2 | `publishManyEventsBulk`: bulk outbox → N sequential chunks + per-chunk `updateMany` | **DONE** |
| B.7.3 | Tests outbox bulk relay (207→chunk FAILED, `shouldUseBulkRelay`, sequential retry) | **DONE** |
| B.7.4 | `.env.example` + local docs aligned | **PARTIAL** |

**B.5 Andreani (design + status):**

| ID | Qué | Status |
|----|-----|--------|
| B.5.1–B.5.5 | `UNMATCHED` status + partial index; MATCH lookup by `idAndreani`; matrix integration; MATCH persist + outbox; NO MATCH inbox-only | **DONE** |
| B.5.6 | Tests MATCH/UNMATCHED/idempotency/outbox | **PARTIAL** — falta `ambiguous_match` explícito |
| B.5.7 | Replay UNMATCHED | **v2 — out of MVP** |
| B.5.8 | Shared `OollProcessingService` (matrix + watermark + outbox); thin per-carrier consumers | **DONE** |
| B.5.9 | Watermark failure-context delta — re-emit failures with different `motivo`/comentario; timeline N×`FAILED`→`DELIVERED` | **DONE** |
| B.5.10 | Orchestrator + seeders cover prod shapes (`EnvioEntregado`, `Visita`, `GestionTelefonica`, unmapped noise) | **DONE** |
| B.5.11 | `Visita` + text-only `"Entregado"` (no `motivo` 99) — **analysis debt**, do not change mapping before tracing legacy | **OPEN** |

B.5.9 emission rule: emit on OOLL failure when (1) state delta vs watermark, **or** (2) failure-context fingerprint delta (`reasonCode`+`reasonLabel`+`comment`, normalized; Andreani `*` omitted). Projector: no contract change — append timeline by `externalEventId`; `resolveCurrentMilestone` unchanged.

**B.8 last-mile terminals (`domain-model.md` §2.1, §4.3):** only `DELIVERED` closes the v1 cycle. `CANCELLED` is a domain terminal but manual — out of active scope (B.8.5 not prioritized). `FAILED` always projects (macro + `subStatus` + `deliveryFailureContext` + timeline), never freezes the cycle; retries → `DELIVERED` allowed. Failure payload: `stateChange.canonicalStatus: FAILED` + `deliveryFailureContext`, **no** `milestoneKey`. B.8.1–B.8.3, B.8.6–B.8.7 DONE; B.8.4 `isTerminal` infra DONE (do not use `milestoneKey: FAILED` in v1).

### Fase C — Projector read model

| ID | Task | Status |
|----|------|--------|
| C.1 | Migration: `current_milestone` on `delivery_history_views` | **DONE** |
| C.2 | Milestone catalog HTTP + mock (`MOCKS_ENABLED`); tabla `milestones` + seeder | **PARTIAL** — endpoint ✅ (static `MILESTONE_CATALOG_SEED_ROWS`); persistence table pending/decision |
| C.3 | Projection mapper sets `current_milestone` from `milestoneKey` | **DONE** |
| C.4 | `PackageEvent.shipmentId` = active trail + real `segments[]` per projection | **NOT STARTED** |
| C.5 | Block 11: populate `delivery_history_search_index` in projection tx | **DONE** |
| C.6 | Block 11 read + Block 12: `GET /shipments?q=&courier=&deliveryType=&milestone=` | **DONE** |
| C.7 | Terminals 5 keys: Zod, `resolveCurrentMilestone`, tests | **DONE** |
| R2 | Detail v2: normalized DTO + regenerated mocks | **DONE** (validate local: `MOCKS_ENABLED=true` + list → detail) |

### Fase D — Dev tooling & validation

| ID | Task | Status |
|----|------|--------|
| D.1 | `POST /ingestor/journeys/:journeyId/reconcile/chunk` | **NOT STARTED** |
| D.2 | Local reconciliation loop script | **NOT STARTED** |
| D.3 | Align `.env.example` (API, projector, fenix-local) to local defaults | **PARTIAL** — prod URL `/api/event` ✅; local docs + bulk env = B.7.4 |
| D.5 | `simulateAndreaniEvents.js` — Fenix consume simulator | **DONE** |

> **D.1/D.2 context:** an alternative **non-LPC** reconciliation via a TMS lookup API (gated, time-windowed, anti-DDOS) was specced for packages that never get an LPC journey. Only build if non-LPC/non-warehouse reconciliation stays in v1 closure scope; otherwise defer with §5.

---

## 7. Search performance (SP.1 / SP.2)

Read-only optimization of `GET /delivery-history/shipments`; **no HTTP contract / pagination semantics change** (single endpoint, `q` 3–64 chars else 400, `LIKE prefix%` on normalized `field_value`, exact filtered `total`, `ORDER BY updated_at DESC`, index write only in projection tx).

| ID | Change | Status |
|----|--------|--------|
| SP.1 (P1.1) | Partial indexes on `delivery_history_views` (`courier`, `delivery_type`, `current_milestone` + `updated_at DESC` where `is_listable`) — migration `20260629140000_add_listable_view_partial_indexes` | **CODED** |
| SP.1 (P1.2) | Single SQL with `COUNT(*) OVER()` + minimal list-row mapping (`toDomainFromListRow`), no relation `include` | **CODED** |
| SP.1 (P1.3) | List without `q`: explicit `select` (no `include`) | **CODED** |
| SP.1 (P1.4) | Document reference `EXPLAIN` plans (95k) | **PENDING** |
| SP.1 gate | Re-run `npm run search-load` @ 250k, archive summary, compare p50/p95 same hardware/seed | **PENDING (user-run)** |
| SP.2 | P2: denormalized `search_index` columns / Block 13 partitioning + TTL | **DEFERRED** |

Acceptance SP.1 = relative improvement ≥30% in stress p95 vs prior baseline on the **same** machine + seed. Out of scope: approximate/capped `total`, keyset pagination, GIN/full-text (contradicts the prefix B-tree strategy). Benchmark procedure + dual-dataset Postgres setup: `testing-orchestrator/README.md`.

---

## 8. Decision log

| ID | Decision | Rationale |
|----|----------|-----------|
| D-01 | **Single source:** `plan.md` only carries status + design; code is the implementation source | Reduce cognitive load; avoid drift |
| D-02 | **Andreani `deliveryFailureContext` v1:** passthrough wire fields (`motivo`→`reasonCode`, `motivoDesc`→`reasonLabel`, `subMotivoDesc`→`comment`); `*` omitted | Matches `andreani-webhook` + adapter; no semantic slug in v1 |
| D-03 | **B.5.9 failure fingerprint:** normalized `reasonCode`+`reasonLabel`+`comment` on watermark; emit on context-only delta | Multi-attempt failed timeline before `DELIVERED` |
| D-04 | **Semantic `reasonCode` (e.g. `DESTINATARIO_AUSENTE`):** deferred post-v1 | Needs catalog + adapter + watermark migration |
| D-05 | **SP.1:** P1.1–P1.3 implemented; acceptance = Artillery gate @ 250k archived | Iterable until v1 benchmark recorded |
| D-06 | **E2E/search evidence:** reproducible runs, not copied session IDs | See § Validation evidence |
| D-08 | **`fechaHora`:** Andreani business event time; `Z` = UTC; broker timestamp = ingest latency | API uses `new Date(fechaHora)` as `occurredAt` |
| D-09 | **`idCliente`:** optional/empty in prod; stored in inbox raw; **not** used for MATCH (MATCH by `idAndreani`) | Aligns with `andreani-webhook` |
| D-10 | **Status lookup keys:** only `evento`, `motivo`, `subMotivo` — not desc fields | Desc → `deliveryFailureContext` only |
| D-11 | **Unmapped Andreani:** inbox `PROCESSED`, no events, no projection (v1) | e.g. `ExpedicionHojaDeRutaDeViaje` |
| D-13 | **`Visita` + text `"Entregado"` without `motivo` 99:** unmapped today — **analysis before fix** (B.5.11) | Trace legacy TMS/Delivery Manager first |
| D-14 | **Non-warehouse path:** model must not hardcode warehouse as the only entry/bootstrap (§5) | EnvíoPack routes high-volume parcels straight to OOLL |
| D-15 | **Draft identity `EP.1`:** cada `EP_READY` crea draft propio con key `EP_BASE + receivedDateEpochMs`; `EP_BASE` searchable no-unique (no `PackageIdentifierLink`); promueve `readyToCollectAt` del draft `OPEN` más **nuevo**. Colisión canónica `EP_BASE`/`EP_FULL` reutilizado = diferida | Acepta duplicados sin tocar identidad canónica; ver `duplicated-ep-tracking.md` |
| D-16 | **`readyToCollectAt` más nuevo (no el más viejo):** la promoción toma el draft `OPEN` con `receivedDateEpochMs DESC` (última intención conocida) | Caso anómalo excluido de KPI igual; decisión explícita, no implícita |
| D-17 | **`FIRST_MILE` leg en `EP_READY` (PENDIENTE decidir):** quitar `ensureFirstMileShipmentLink` de `consumeEpReadyToCollectOrders` es **global** (toca A.1 DONE), no draft-only. Alternativa: mover creación a materialización | No bakear el cambio bajo "identidad de drafts"; verificar ripple en `applyOeLaunchTrailTransitions` |

---

## 9. Do Not Do

- Do not modify `framework/` or per-project `AGENTS.md` standards (framework exception: Fenix queue/outbox for B.7, already applied).
- Do not project `DRAFT_BASE`.
- Do not emit `PACKAGE_NEW_MILESTONE` from WMI sync or journey chunk; do not emit on WMS classification (persist only).
- Do not create `LAST_MILE` shipment in Block 3 chunk.
- Do not treat the 11 `*At` fields as 11 UI milestones.
- Do not insert `outbox_ingestor` or upsert watermark from gate use cases (outbox activates post-return via `EventPublisherInterceptor` → `FenixOutboxProducer`).
- Do not update projection watermark post-`SENT` (claim in producer tx with outbox `PENDING`).
- Do not `Promise.all` / fixed-delay between Fenix bulk chunks; do not bulk-relay `eventId` retries.
- Do not project `EP_READY` drafts, give them an `EP_BASE` `PackageIdentifierLink`, or emit from `consumeWmsPackageClassification` when adding the `EP.1` draft fallback (stays persist-only).

---

## 10. Validation evidence

Artifacts live under `testing-orchestrator/runs/<timestamp>/` (`report.json`, Artillery summaries). Rules:

1. **Do not cite** run IDs/timestamps from another machine as acceptance proof.
2. **If no fresh run exists** after material changes (watermark, Andreani buckets, search repo, migrations), **warn the user** before claiming E2E or search-perf acceptance.
3. **After relevant changes,** re-run `npm run e2e` and/or `npm run search-load` (or `npm run validate`), archive output, compare p50/p95 to the previous baseline on the **same** hardware + seed count.
4. **Orchestrator DB:** `PROJECTOR_DATABASE_URL` must equal the projector `POSTGRES_DATABASE_URL`.

Full runbook, step gotchas, Andreani buckets, dual-dataset setup, SQL validation: `testing-orchestrator/README.md`.

---

## 11. Doc map

| Doc | Role |
|-----|------|
| **`plan.md`** | **Single source** — status + full context + design decisions (this file) |
| `domain-model.md` | Domain authority (milestones, tramos, gates, watermark) |
| `logistics.md` | Business / physical context + real wire shapes; **non-warehouse scalability** source |
| `andreaniEvents.md` | Andreani production wire samples (reference; not exhaustive) |
| `testing-orchestrator/README.md` | E2E pipeline + manual runbook + SQL validation + search load |
| `shipments-history-api/AGENTS.md`, `shipments-history-projector/AGENTS.md` | Per-project coding standards — load before writing code |
| `fenix-local/README.md` | Local Fenix emulator |

Implemented block-level design (API Blocks 1–8, projector Blocks 1–13, R1/R2) now lives in **code + tests**; this file keeps the decisions and contracts that are not obvious from code.
