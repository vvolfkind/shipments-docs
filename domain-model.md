# Domain Model — Milestones, Tramos, Estados y Gates

**Status:** FINAL (2026-06-26) — last-mile terminals: only **`DELIVERED`** closes v1 cycle; **`FAILED`** projects without terminal; **`CANCELLED`** manual out of active scope.  
**Authority:** Este documento es la fuente de verdad para definiciones de dominio. Si `blocks-api.md`, `blocks-projector.md` o `logistics.md` contradicen este archivo, **este archivo gana** hasta que se actualicen explícitamente.

**Alcance:** `shipments-history-api/`, `shipments-history-projector/`, contrato `PACKAGE_NEW_MILESTONE`.

---

## 1. Cuatro capas (no confundir)

| Capa | Qué es | Quién decide | Dónde vive | ¿Va al projector? |
|------|--------|--------------|------------|-------------------|
| **A. Timestamps operativos** | Evidencia temporal de pasos logísticos (`readyToCollectAt`, `classifiedAt`, …) | API al ingestar cada fuente | API DB `package_milestones`; projector replica los incluidos en `operationalTimestamps` del evento | Solo como **datos empaquetados** en un gate; nunca como trigger individual |
| **B. Pasos intermedios / timeline** | Eventos append-only con contexto (`PackageEvent`) | API | API DB; projector replica en timeline de detalle | Sí, como timeline; **no** como dimensión de filtro |
| **C. Estado macro + sub-estado operativo** | `canonicalStatus` + `subStatus` para diagnóstico | API (diccionario carrier → par estandarizado) | `Package`, `Shipment`, eventos | `stateChange` en cada emisión; **subStatus nunca es filtro UI** |
| **D. Milestones de proyección (`MilestoneKey`)** | Puntos estratégicos que consolidan información y abren/actualizan read model listable | API (projection gates) | Projector `delivery_history_views.current_milestone` + catálogo Block 12 | Sí — **solo** cuando se cumple un gate |

**Regla de oro:** Los 11 campos `*At` en `PackageMilestones` son **timestamps operativos (capa A)**, no milestones de proyección (capa D). Un package puede tener los 11 timestamps en API y solo 2–3 `MilestoneKey` en projector.

### 1.1 Catálogo timestamps operativos (`PackageMilestones`)

Persistencia en API (`package_milestones`); projector replica solo los incluidos en `milestoneDeltas` / `operationalTimestamps` del evento.

| Campo | Fase | Persist API | Emite projector | writePolicy |
|-------|------|-------------|-----------------|-------------|
| `readyToCollectAt` | Primera milla | Block 2 | Gate `WAREHOUSE_RECEIVED` (bundle) | set-once |
| `collectedAt` | Primera milla | **TBD consumer** | Gate `WAREHOUSE_RECEIVED` (bundle) | set-once |
| `journeyAnnouncedAt` | Primera milla | Block 3 | Gate `WAREHOUSE_RECEIVED` (bundle) | set-once |
| `oeLaunchAt` | Frontera tramo | Block 6.3 | Gate `WAREHOUSE_RECEIVED` | set-once |
| `classifiedAt` | Operativa CD | Block 6.4 | Gate `LAST_MILE_DISPATCHED` (bundle) | set-once |
| `dispatchedAt` | Operativa CD | Block 6.5 | Gate `LAST_MILE_DISPATCHED` | set-once |
| `pickedUpByCarrierAt` | OOLL | Block 5 | Emisión incremental OOLL (sin `MilestoneKey`) | set-once |
| `consolidatedAtCarrierAt` | OOLL | Block 5 | Emisión incremental OOLL | set-once |
| `inTransitAt` | OOLL | Block 5 | Emisión incremental OOLL | **max**(`occurredAt`) |
| `deliveredAt` | OOLL | Block 5 | Gate `DELIVERED` | set-once |

En payload `PACKAGE_NEW_MILESTONE`, `milestoneDeltas` transporta timestamps operativos empaquetados (alias de `operationalTimestamps` hasta refactor).

---

## 2. Estado macro de negocio

Desde herramientas actuales de negocio, un package que fue anunciado listo para colectar **ya está en tránsito** a nivel macro. La primera milla, el viaje al CD y la operativa interna del warehouse son **fases operativas dentro del mismo macro-estado**, no estados de producto independientes.

### 2.1 `canonicalStatus` v1 (macro)

| Valor | Cuándo | Notas |
|-------|--------|-------|
| `IN_TRANSIT` | Ciclo operativo normal (warehouse + checkpoints OOLL sin cierre) | Default hasta terminal |
| `DELIVERED` | Entrega confirmada por OOLL | **Terminal éxito** v1 — macro = `MilestoneKey` |
| `FAILED` | Fallo de entrega OOLL | Macro transitorio — **proyecta siempre**; **no** cierra ciclo; grilla mantiene último progreso |
| `CANCELLED` | Cancelación | Terminal dominio — flujo manual; **fuera de scope v1 activo** |
| `CREATED` / internos | Solo draft API (`DRAFT_BASE`) | **No** proyectable |

Warehouse (`oeLaunchAt`…`dispatchedAt`) y checkpoints OOLL no-fallidos ocurren con macro **`IN_TRANSIT`**. Tras un fallo OOLL reintentable, macro pasa a **`FAILED`** hasta entrega (`DELIVERED`) u otro checkpoint que negocio defina. La fase warehouse se expresa por **tramo**, **timestamps operativos** y **`MilestoneKey`**, no por un `canonicalStatus` aparte tipo `WAREHOUSE_PROCESSING`.

### 2.2 Sub-estado operativo (`subStatus`)

- API traduce raw carrier/WMS → par `(status, subStatus)` estandarizado.
- Sirve para **timeline y diagnóstico** en detalle.
- **No** es dimensión de filtro en grilla (can of worms).
- **No** abre `MilestoneKey` por sí solo.
- Ejemplos: `Recepcionado por OOLL`, `Entrega fallida`, `CLASSIFIED` (interno WMS).

### 2.3 Estados intermedios (pasos operativos)

Pasos que negocio hoy mezcla con "estado" pero ingeniería trata como **evidencia**, no como milestone de proyección:

| Fase | Pasos operativos (timestamps) | Milestone de proyección |
|------|------------------------------|-------------------------|
| Primera milla | `readyToCollectAt`, planificación/colecta (`collectedAt`, journey WMI) | **Ninguno** hasta gate warehouse |
| Llegada CD | `journeyAnnouncedAt`, `oeLaunchAt` | `WAREHOUSE_RECEIVED` |
| Operativa CD | `classifiedAt` (y metadata: LPN, operario) | **Ninguno** — solo persist API |
| Salida CD | `dispatchedAt` | `LAST_MILE_DISPATCHED` |
| Última milla OOLL | `pickedUpByCarrierAt`, `consolidatedAtCarrierAt`, `inTransitAt` | Proyección **obligatoria** (capa B+C); macro `IN_TRANSIT`; sin nuevo `MilestoneKey` de progreso |
| Entrega | `deliveredAt` | `DELIVERED` (macro + milestone) |
| Entrega fallida OOLL | — | Proyección **obligatoria**: macro `FAILED` + `subStatus` + `deliveryFailureContext`; **sin** `milestoneKey`; **no** terminal |

---

## 3. Tramos logísticos (`trailType`)

### 3.1 Modelo binario v1 (FINAL)

| Tramo | `trailType` | Inicio | Fin | Qué incluye |
|-------|-------------|--------|-----|-------------|
| **Primera milla** | `FIRST_MILE` | `ep-ready-to-collect-orders` (o primer evento first-mile) | Llegada física al CD propio (`oeLaunchAt`) | Listo para colectar, viaje LPC, colecta |
| **Última milla** | `LAST_MILE` | `oeLaunchAt` | `deliveredAt` (o cierre operativo v1) | **Todo** post-llegada CD: operativa warehouse + carrier OOLL + entrega |

**Decisión cerrada:** No usar `WAREHOUSE_INTERNAL` como tercer tramo en v1. La operativa dentro del CD es fase de **`LAST_MILE`**, no tramo separado.

### 3.2 Entidades

- **`Shipment`** = contrato/tramo con operador (`trailType`, `carrierCode`, `trackingNumber`).
- **`PackageShipment`** = membership package↔tramo con `enteredAt` / `exitedAt`.
- **`Journey`** = viaje LPC de colecta (entidad operativa WMI). **No** reemplaza `Shipment`; correlaciona primera milla.

### 3.3 Fronteras de tramo

```
ep-ready ──► [ FIRST_MILE: EnvioPack colecta ] ──► oeLaunchAt ──► [ LAST_MILE: CD + OOLL + entrega ] ──► deliveredAt
              enteredAt                              exitedAt/enteredAt
```

En gate `OE_LAUNCH`:
1. Cerrar `PackageShipment` de `FIRST_MILE` (`exitedAt = oeLaunchAt`).
2. Abrir `PackageShipment` de `LAST_MILE` (`enteredAt = oeLaunchAt`).
3. En HD: el `Shipment LAST_MILE` carrier (Andreani tracking) se vincula **desde este gate**, no antes.

### 3.4 Reglas API (persistencia)

| Acción | Permitido | Prohibido |
|--------|-----------|-----------|
| Block 2 `ep-ready` | Crear `Shipment FIRST_MILE` + link | Emitir al projector |
| Block 3 chunk WMI | Persistir `journeyAnnouncedAt`, links `LAST_MILE` **identifier** | Crear `Shipment LAST_MILE` |
| Block 6.3 `OE_LAUNCH` | Cerrar first mile, abrir last mile, emitir gate | — |
| Block 6.4 clasificación | Persist `classifiedAt` + metadata | Emitir `PACKAGE_NEW_MILESTONE` |
| Block 6.5 despacho físico | Persist `dispatchedAt`, emitir gate | Emitir en `DISPATCH_CONTAINER_INTENT` |

### 3.5 Lectura por tramo (projector, sin refactor futuro)

Detail ya expone `segments[]` por `Shipment.trailType`. Con tramos bien acotados, frontend puede:
- Filtrar timeline/milestones con corte en `oeLaunchAt`.
- Mostrar tracking/courier por segmento.
- Agrupar operativa CD bajo segmento `LAST_MILE`.

---

## 4. Milestones de proyección (`MilestoneKey`)

### 4.1 Catálogo cerrado v1

| `MilestoneKey` | Label UI (es) | Gate / trigger | `emissionTrigger` |
|----------------|---------------|----------------|-------------------|
| `WAREHOUSE_RECEIVED` | Recibido en depósito | `WM2_RS_MAIN_OE_LAUNCH` | `OE_LAUNCH` |
| `LAST_MILE_DISPATCHED` | Despachado a última milla | `CONTAINER_PACKAGES_SHIPPED` | `WAREHOUSE_DISPATCH` |
| `DELIVERED` | Entregado | OOLL entrega confirmada | `OOLL_DELIVERED` |
| `FAILED` | Entrega fallida | Macro OOLL — **no** gate terminal v1 | — |
| `CANCELLED` | Cancelado | Manual / `Anulada` — fuera scope v1 activo | `MANUAL_CANCEL` / `OOLL_CANCELLED` |

**Grilla (`current_milestone`):** valor = último `MilestoneKey` aplicado según reglas §4.3 (no el último timestamp operativo). Macro `FAILED` reintentable **no** avanza milestone de grilla.

**Listabilidad:** MVP warehouse — primer gate listable = `WAREHOUSE_RECEIVED`. Cancel/`Anulada` **antes** de ese gate → no aparece en grilla. Futuros flujos sin warehouse pueden usar otro entry gate (`plan.md` § B.8).

### 4.3 Reglas de transición (`current_milestone` y macro)

**Progreso** (orden estricto forward-only): `WAREHOUSE_RECEIVED` → `LAST_MILE_DISPATCHED`.

**Terminales v1 (cierre ciclo):** solo **`DELIVERED`**.

| Regla | Descripción |
|-------|-------------|
| Macro `FAILED` → proyección | Siempre; **no** avanza `current_milestone` |
| Macro `FAILED` → `DELIVERED` | Permitido (reintentos OOLL) |
| `DELIVERED` → `FAILED` | Prohibido |
| `CANCELLED` | Fuera scope v1 activo (manual) |

Implementación projector: `plan.md` **C.7** ✅.

### 4.2 Qué empaqueta cada gate

#### Gate 1 — `WAREHOUSE_RECEIVED` (`OE_LAUNCH`)

Primer contacto projector. **Consolida primera milla** en un único gate (timestamps + cierre tramo `FIRST_MILE`):

- `readyToCollectAt`, `collectedAt` (si existen en API), `journeyAnnouncedAt`, `oeLaunchAt`
- No hay proyección projector antes de este gate para el camino warehouse

```json
{
  "milestoneKey": "WAREHOUSE_RECEIVED",
  "emissionTrigger": "OE_LAUNCH",
  "occurredAtUtc": "<oeLaunchAt>",
  "stateChange": {
    "canonicalStatus": "IN_TRANSIT",
    "subStatus": null,
    "lastOccurredAt": "<oeLaunchAt>"
  },
  "operationalTimestamps": {
    "readyToCollectAt": "<si existe en API>",
    "collectedAt": "<si existe>",
    "journeyAnnouncedAt": "<desde Block 3>",
    "oeLaunchAt": "<trigger>"
  },
  "trailTransitions": {
    "closeFirstMile": { "exitedAt": "<oeLaunchAt>" },
    "openLastMile": { "enteredAt": "<oeLaunchAt>", "carrierCode": "...", "trackingNumber": "..." }
  },
  "metadata": { "projectionBootstrap": true, "journeyId": "...", "itemType": "HD|SPU" }
}
```

**Nota de contrato:** `milestoneDeltas` en código actual se mantiene como alias de `operationalTimestamps` hasta refactor; semántica = timestamps empaquetados, no "milestone por campo".

#### Gate 2 — `LAST_MILE_DISPATCHED` (`CONTAINER_PACKAGES_SHIPPED`)

Consolida operativa CD. **No** hay gate separado por clasificación.

```json
{
  "milestoneKey": "LAST_MILE_DISPATCHED",
  "emissionTrigger": "WAREHOUSE_DISPATCH",
  "occurredAtUtc": "<dispatchedAt>",
  "operationalTimestamps": {
    "classifiedAt": "<set-once desde WMS>",
    "dispatchedAt": "<trigger>"
  },
  "warehouseContext": {
    "containerLpn": "<LPN contenedor>",
    "classifiedByUserId": "<operario WMS>"
  }
}
```

#### Gate 3+ — Última milla OOLL (proyección obligatoria)

**Regla cerrada:** todo aviso OOLL reconciliado que cambie `canonicalStatus`, `subStatus` y/o timestamp operativo **debe** emitir `PACKAGE_NEW_MILESTONE` al projector — **solo si hay MATCH** de package en API (`blocks-api.md` §5.8). Sin MATCH: persistir inbox `UNMATCHED`; sin emisión (v1 forward-only).

- Solo tramo **`LAST_MILE`** (post-`oeLaunchAt`) salvo flujos non-warehouse futuros.
- Matriz Andreani raw → par → timestamp: **§4.4**.
- Checkpoints OOLL (admisión, consolidación, distribución): macro `IN_TRANSIT`; sin avance de milestone de progreso.
- **`MilestoneKey` de progreso** no avanza salvo gates warehouse; único cierre v1: **`DELIVERED`**. Fallos OOLL: macro `FAILED` sin `milestoneKey`.
- Projector debe recibir `stateChange` en cada aviso relevante.

Ejemplo checkpoint OOLL (sin `MilestoneKey`):

```json
{
  "operationalTimestamps": { "pickedUpByCarrierAt": "2026-06-12T10:15:00.000Z" },
  "stateChange": {
    "canonicalStatus": "IN_TRANSIT",
    "subStatus": "Recepcionado por OOLL",
    "lastOccurredAt": "2026-06-12T10:15:00.000Z"
  }
}
```

**Entrega fallida OOLL:**
- **Sí** proyecta (sin timestamp nuevo, **sin** `milestoneKey`).
- Macro **`FAILED`** — **no** terminal; ciclo puede continuar hasta `DELIVERED`.

```json
{
  "stateChange": {
    "canonicalStatus": "FAILED",
    "subStatus": "Entrega fallida",
    "lastOccurredAt": "2026-06-13T16:45:00.000Z"
  },
  "deliveryFailureContext": {
    "reasonCode": "50",
    "reasonLabel": "Destinatario ausente",
    "comment": "Domicilio cerrado"
  }
}
```

**Andreani v1 wire mapping** (`andreani-webhook` pass-through → `andreaniOoll.adapter.ts`):

| Raw Andreani | Projected field | Notes |
|--------------|-----------------|-------|
| `motivo` | `reasonCode` | Carrier code string (e.g. `"50"`) |
| `motivoDesc` | `reasonLabel` | Catalog label; omitted if `*` or empty |
| `subMotivoDesc` | `comment` | Courier / sub-reason detail; omitted if `*` or empty |
| `subMotivo` | — | Status-mapping lookup only; not sent to projector |

Include `deliveryFailureContext` when the carrier sends failure data (`reasonCode` required when failure is typed).

| Field | Required | Notes |
|-------|----------|-------|
| `reasonCode` | Yes (if failure) | v1 Andreani: `motivo`; normalized semantic slug deferred post-v1 |
| `reasonLabel` | No | v1 Andreani: `motivoDesc` when not `*` |
| `comment` | No | v1 Andreani: `subMotivoDesc` when not `*` |

API persiste en `PackageEvent.payload` + incluye en evento projector. Projector muestra en timeline de detalle (última milla).

#### Gate cierre — `DELIVERED` (único terminal v1)

```json
{
  "milestoneKey": "DELIVERED",
  "emissionTrigger": "OOLL_DELIVERED",
  "operationalTimestamps": { "deliveredAt": "<trigger>" },
  "stateChange": { "canonicalStatus": "DELIVERED", "subStatus": "Entregada" }
}
```

#### `CANCELLED` (fuera scope v1 activo)

Andreani `Anulada` o cancel manual — código existe; no priorizado v1. Ver `plan.md` B.8.5 OUT OF SCOPE.

```json
{
  "milestoneKey": "CANCELLED",
  "emissionTrigger": "OOLL_CANCELLED",
  "stateChange": {
    "canonicalStatus": "CANCELLED",
    "subStatus": "Anulada",
    "lastOccurredAt": "<trigger>"
  }
}
```

`cancellationContext` (motivo, actor): opcional — no bloquea emisión.

### 4.4 Andreani OOLL v1 — wire shapes and mapping

Resolution uses standardized `(status, subStatus)` via `CarrierStatusMapping`. Lookup keys are built from **`evento`**, **`motivo`**, **`subMotivo` only** — description fields do not participate. Only **`LAST_MILE`** trail.

**Production reference samples:** `andreaniEvents.md` (curated; not exhaustive).

**Timestamps:** `fechaHora` is the carrier business event time. Values with suffix **`Z`** are UTC (ISO 8601). API sets `occurredAt = new Date(fechaHora)`. Queue/broker ingest time (e.g. Redpanda) reflects pipeline latency, not business time.

**Optional wire fields:** `idCliente` may be empty in production; stored in inbox `raw_payload` but **not used** for package MATCH (tracking id = `idAndreani`).

#### Mapped families (catalog + prod samples)

| `evento` (and codes) | Canonical `subStatus` | Timestamp / notes | Projects |
|----------------------|----------------------|-------------------|----------|
| `Admision`, `AltaAutomatica`, … | Recepcionado por OOLL | `pickedUpByCarrierAt` | Yes (incremental) |
| `EnvioConsolidado` | En tránsito / Consolidado | `consolidatedAtCarrierAt` | Yes |
| `Distribucion`, `EnvioDespachado` | En tránsito / En distribución | `inTransitAt` | Yes |
| **`Visita`** + `motivo`/`subMotivo` (e.g. `26,22`) | Entrega fallida | — | Yes: macro `FAILED` + `deliveryFailureContext` |
| **`GestionTelefonica`** + codes 50/51/… | Entrega fallida | — | Yes (same as above; common in E2E fixtures) |
| **`EnvioEntregado`** (+ optional `motivo` 99) | Entregada | `deliveredAt` | Yes + `DELIVERED` |
| `GestionTelefonica` + code 99 | Entregada | `deliveredAt` | Yes + `DELIVERED` |
| `Anulada` | Anulada | — | B.8.3; out of active v1 scope |

Retryable failures: no new operational timestamp; macro **`FAILED`**; include `deliveryFailureContext` when raw carries codes/descriptions.

#### Unmapped / PROCESSED-only (v1 — by design)

No `CarrierStatusMapping` match → inbox **`PROCESSED`**, no `package_events`, no projector emission. Examples from prod:

| Wire shape | Why unmapped |
|------------|--------------|
| `ExpedicionHojaDeRutaDeViaje` | Operational route sheet; not in catalog |
| `Visita` with **empty** `motivo`/`subMotivo` | No key `Visita` in catalog (only compound keys) |
| `Visita` + free-text desc only (e.g. `"No responde llamado"`) | Desc fields not in lookup |

#### Open analysis (debt — plan.md B.5.11)

**`Visita` + `motivoDesc`/`subMotivoDesc` = `"Entregado"` without `motivo` 99:** semantically delivery, but lookup fails today. **Do not change mapping until** legacy TMS / Delivery Manager behavior is traced (how downstream interpreted this shape). Task owner: ops + API.

**E2E/orchestrator note:** orchestrator + projector seeders cover **`Visita`** (coded failure), **`EnvioEntregado`** (delivery), **`GestionTelefonica`** (multi-failure B.5.9), and unmapped noise (**B.5.10** ✅). **B.5.11** (`Visita` + text-only `"Entregado"`) remains unmapped by design until legacy analysis.

**Other carriers (future):** same pattern — map status → operational timestamp; decide incremental vs terminal gate.

---

## 5. Projection gates (resumen)

| # | Gate | Emite `PACKAGE_NEW_MILESTONE` | Avanza `MilestoneKey` | Persist API sin emitir |
|---|------|------------------------------|----------------------|------------------------|
| — | Block 2 ep-ready | No | — | `readyToCollectAt`, `FIRST_MILE` shipment |
| — | Block 3 chunk | No | — | `journeyAnnouncedAt`, identifiers |
| — | Block 6.4 classification | **No** | — | `classifiedAt`, LPN, operario |
| 1 | `OE_LAUNCH` | **Sí** (bootstrap) | `WAREHOUSE_RECEIVED` | `oeLaunchAt` |
| 2 | `CONTAINER_PACKAGES_SHIPPED` | **Sí** | `LAST_MILE_DISPATCHED` | `dispatchedAt` |
| 3 | OOLL checkpoints | **Sí** (obligatorio) | — (progreso / macro `IN_TRANSIT`) | timestamps según matriz |
| 3b | OOLL entrega fallida | **Sí** | — (macro `FAILED`, sin milestone) | `deliveryFailureContext` |
| 4 | OOLL entregada | **Sí** | `DELIVERED` | `deliveredAt` |

**Watermark de proyección (B.2):** ver §5.1. Emisión = gate satisfecho ∧ delta vs watermark ∧ identidad proyectable.

### 5.1 Projection watermark (`package_projection_watermarks`)

**Schema API:** ✅ `PackageProjectionWatermark` → `package_projection_watermarks`.

**Implementación watermark:** diseño en este §; **status y archivos:** `plan.md` B.2.

**Propósito:** ledger de lo que API **reclamó** proyectar por bulto — no es el read model del projector.

| Campo | Rol |
|-------|-----|
| `lastMilestoneKey` | Último `MilestoneKey` incluido en una emisión |
| `projectedTimestamps` | JSON `{ readyToCollectAt: "…", oeLaunchAt: "…" }` — keys ya enviadas en `milestoneDeltas` / `operationalTimestamps` |
| `lastProjectedCanonicalStatus` / `lastProjectedSubStatus` | Last projected `stateChange` (OOLL without `MilestoneKey` advance) |
| `lastProjectedFailureReasonCode` / `lastProjectedFailureReasonLabel` / `lastProjectedFailureComment` | Last projected `deliveryFailureContext` fingerprint (B.5.9) |
| `lastProjectedAt` | Timestamp of the last claimed emission trigger |

**Dos responsabilidades (no mezclar):**

| Pregunta | Capa | Cuándo |
|----------|------|--------|
| ¿Hay delta y qué va en el payload? | Gate use case (+ lectura watermark) | Sync, **antes** del `return { event }` |
| ¿Reclamo proyección + encolo outbox? | `FenixOutboxProducer` (vía interceptor) | **Post-return** del use case |
| ¿Llegó a Fenix? | `outbox_ingestor` (`PENDING` → `SENT` / `FAILED`) | Relay en producer (existente) |

**Flujo cerrado (respeta patrón outbox post-return):**

```
Gate use case (ConsumeOeLaunch / ConsumeWmsContainerPackagesShipped / OOLL)
  1. persist operativo (repos / SQL bulk — como hoy)
  2. leer watermark (bulk per-package)
  3. calcular delta → buildProjectionEvents() solo para bultos con delta
  4. return { event: EventResponse | EventResponse[] }   // sin writes a outbox ni watermark

EventPublisherInterceptor (post-return)
  → FenixOutboxProducer.publishManyEvents / publishManyEventsBulk

FenixOutboxProducer (activación outbox)
  BEGIN TX
    bulk upsert package_projection_watermarks   // claim desde payload de cada event
    createManyAndReturn outbox_ingestor PENDING
  COMMIT
  produce Fenix (B.7: chunks secuenciales → `/api/event/bulk` cuando N≥2 sin retry; hoy per-event `/api/event`)
  updateMany outbox SENT / FAILED
```

**Reglas de persistencia:**

1. **Lectura** watermark en el gate use case (o servicio de aplicación invocado desde él) **antes** de `buildProjectionEvents`. Si no hay delta → `return` sin `event`.
2. **Escritura** watermark (upsert per-package) **solo** en `FenixOutboxProducer`, en la **misma transacción** que `outbox_ingestor` `PENDING`, **antes** del relay Fenix. El use case **no** inserta outbox ni watermark.
3. **No** actualizar watermark post-`SENT`. Reintentos: outbox `PENDING`/`FAILED` + `event.payload.eventId` (path retry existente); mismo payload y `externalEventId`.
4. Re-ingest duplicado: watermark ya reclamado + warehouse `appliedPackageIds`/set-once → no nuevo `{ event }`; si igual llegara un `{ event }`, el upsert en producer debe ser idempotente (`ON CONFLICT` / skip si ya reclamado).
5. Persist operativo del gate (ej. `applyOeLaunchForJourney`) commitea **antes** del return; atomicidad watermark↔outbox es en el **borde de publicación** (producer), no en la tx del SQL bulk del gate.

**Concurrencia (ventana post-persist, pre-producer):** dos ingests concurrentes pueden pasar lectura de watermark; mitigar con `appliedPackageIds` + set-once en warehouse y upsert condicional en producer.

**Qué ya cubre el código sin watermark (warehouse, replay del mismo gate):**

| Mecanismo | Efecto |
|-----------|--------|
| `package_milestones` set-once (`oe_launch_at`, `dispatched_at`) + `appliedPackageIds` | Re-OE_LAUNCH / re-dispatch no entra a `buildProjectionEvents` si el gate ya aplicó |
| `inbox_events.external_event_id` UNIQUE | Dedup del mensaje Fenix de ingestión |
| `externalEventId` determinístico en payload | Dedup en projector |

**Dónde sí es obligatorio el watermark:**

- Delta multi-field bootstrap and out-of-order gates.
- OOLL (B.5): incremental emissions with same `MilestoneKey`, `subStatus` changes, **repeat delivery failures** with same macro/subStatus but different `deliveryFailureContext` (B.5.9 fingerprint on watermark failure fields).

**Cohortes masivas:** watermark/outbox bulk en `publishManyEventsBulk` ✅; relay Fenix bulk → **`plan.md` § B.7** ✅.

---

## 6. Implementación vs definición

**Checklist y status:** única fuente → **`plan.md`**.

Desvíos abiertos (2026-06-24):

| Tema | Acción |
|------|--------|
| Pendiente implementación | **`plan.md` § Pendiente** |
| `collectedAt` consumer primera milla | TBD |

El resto del camino warehouse + OOLL MVP v1 está alineado con este doc (ver evidencia reproducible en `plan.md` § Validation evidence).

---

## 7. Referencias cruzadas

| Tema | Documento |
|------|-----------|
| Plan operativo + status | **`plan.md`** |
| Onboarding | `prompt.md` |
| E2E local | `local-testing-guide.md` |
| Andreani prod wire samples | `andreaniEvents.md` |
| Fenix bulk relay | `plan.md` § B.7 |
| Terminales última milla | `plan.md` § B.8 · §2.1 |
| Blocks API | `blocks-api.md` |
| Blocks projector | `blocks-projector.md` |
| Flujo negocio | `logistics.md` |
