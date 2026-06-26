# Domain Model — Milestones, Tramos, Estados y Gates

**Status:** FINAL (2026-06-24) — catálogo 5 `MilestoneKey` + terminales `FAILED`/`CANCELLED` (operativa → `plan.md` § B.8)  
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
| `DELIVERED` | Entrega confirmada por OOLL | Terminal éxito; macro = `MilestoneKey` |
| `FAILED` | Fallo de entrega OOLL (reintentable o terminal) | Macro = milestone solo en fallo **terminal**; reintentable: macro `FAILED`, milestone sin avanzar |
| `CANCELLED` | Cancelación manual o carrier `Anulada` | Terminal **absoluto**; macro = `MilestoneKey` |
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
| Entrega fallida reintentable | — (sin timestamp nuevo) | Proyección **obligatoria**: macro `FAILED` + `subStatus` + `deliveryFailureContext`; **sin** `MilestoneKey` |
| Entrega fallida terminal | — | Proyección **obligatoria**: macro `FAILED` + `milestoneKey: FAILED` + `deliveryFailureContext` (subset mapeos carrier — ver spec) |
| Cancelación | — | `CANCELLED` (macro + milestone) solo **post-gate listable**; pre-gate: persist API, sin grilla |

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
| `FAILED` | Entrega fallida | OOLL fallo terminal (subset mapeos) o macro transitorio sin milestone | `OOLL_DELIVERY_FAILED` |
| `CANCELLED` | Cancelado | Manual declarativo; Andreani `Anulada` | `MANUAL_CANCEL` / `OOLL_CANCELLED` |

**Grilla (`current_milestone`):** valor = último `MilestoneKey` aplicado según reglas §4.3 (no el último timestamp operativo). Macro `FAILED` reintentable **no** avanza milestone de grilla.

**Listabilidad:** MVP warehouse — primer gate listable = `WAREHOUSE_RECEIVED`. Cancel/`Anulada` **antes** de ese gate → no aparece en grilla. Futuros flujos sin warehouse pueden usar otro entry gate (`plan.md` § B.8).

### 4.3 Reglas de transición (`current_milestone` y macro)

**Progreso** (orden estricto forward-only): `WAREHOUSE_RECEIVED` → `LAST_MILE_DISPATCHED`.

**Terminales** (reglas especiales):

| Regla | Descripción |
|-------|-------------|
| `FAILED` → `DELIVERED` | **Permitido** — n fallos pueden terminar en entrega |
| `DELIVERED` → `FAILED` | **Prohibido** |
| `*` → `CANCELLED` | Permitido solo post-gate listable |
| `CANCELLED` → `*` | **Prohibido** — terminal absoluto |
| Checkpoint OOLL reintentable | Macro `FAILED`; **no** cambia `current_milestone` de progreso |
| Fallo terminal (mapeo) | Macro + milestone `FAILED` |

Implementación projector: `plan.md` **C.7** ✅. Reglas operativas: `plan.md` § **B.8**.

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
- **`MilestoneKey` de progreso** no avanza salvo gates warehouse; terminales OOLL: `DELIVERED`, `FAILED` (subset), `CANCELLED` (`Anulada`).
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

**Entrega fallida reintentable** (`Entrega fallida`):
- **Sí** proyecta (sin timestamp nuevo, **sin** `MilestoneKey`).
- Macro **`FAILED`** en `stateChange`.
- Incluir `deliveryFailureContext` cuando el carrier lo envíe:

```json
{
  "stateChange": {
    "canonicalStatus": "FAILED",
    "subStatus": "Entrega fallida",
    "lastOccurredAt": "2026-06-13T16:45:00.000Z"
  },
  "deliveryFailureContext": {
    "reasonCode": "DESTINATARIO_AUSENTE",
    "reasonLabel": "Destinatario ausente",
    "comment": "No atiende, timbre sin respuesta"
  }
}
```

**Entrega fallida terminal** (mapeos carrier marcados — fase 2 API): mismo payload + `"milestoneKey": "FAILED"`.

| Campo | Obligatorio | Notas |
|-------|-------------|-------|
| `reasonCode` | Sí (si hay falla) | Motivo pre-establecido del catálogo carrier |
| `reasonLabel` | No | Label normalizado para UI |
| `comment` | No | Texto libre del operador; persistir **solo si llega** |

API persiste en `PackageEvent.payload` + incluye en evento projector. Projector muestra en timeline de detalle (última milla).

#### Gate terminal — `DELIVERED`

```json
{
  "milestoneKey": "DELIVERED",
  "emissionTrigger": "OOLL_DELIVERED",
  "operationalTimestamps": { "deliveredAt": "<trigger>" },
  "stateChange": { "canonicalStatus": "DELIVERED", "subStatus": "Entregada" }
}
```

#### Gate terminal — `FAILED` (subset mapeos)

```json
{
  "milestoneKey": "FAILED",
  "emissionTrigger": "OOLL_DELIVERY_FAILED",
  "stateChange": {
    "canonicalStatus": "FAILED",
    "subStatus": "Entrega fallida",
    "lastOccurredAt": "<trigger>"
  },
  "deliveryFailureContext": { "reasonCode": "..." }
}
```

#### Gate terminal — `CANCELLED`

Andreani `Anulada` (automático) o cancelación manual declarativa. Solo post-gate listable.

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

`cancellationContext` (motivo, actor): opcional v1 — no bloquea emisión.

### 4.4 Mapeo Andreani OOLL v1

Resolución por par estandarizado `(status, subStatus)` vía `CarrierStatusMapping`, no por raw carrier status aislado. Solo tramo **`LAST_MILE`**.

| Andreani raw | Par canónico (`subStatus`) | Timestamp | Emite projector |
|--------------|---------------------------|-----------|-----------------|
| `Admision`, `AltaAutomatica`, `AltaInterna`, `AltaRemota`, `AsignacionACaja` | Recepcionado por OOLL | `pickedUpByCarrierAt` | Sí (incremental) |
| `EnvioConsolidado` | En tránsito / Consolidado | `consolidatedAtCarrierAt` | Sí |
| `Distribucion`, `EnvioDespachado` | En tránsito / En distribución | `inTransitAt` | Sí |
| `GestionTelefonica` + códigos 50/51/… | Entrega fallida | — | Sí: macro `FAILED`, `deliveryFailureContext`; **sin** `milestoneKey` |
| `GestionTelefonica` + códigos 50/51/… (mapeo terminal) | Entrega fallida | — | Sí + `milestoneKey: FAILED` (subset `CarrierStatusMapping` — B.8.4) |
| `Anulada` | Anulada | — | Sí post-gate listable → `CANCELLED`; pre-gate persist only |
| `GestionTelefonica` + código 99 | Entregada | `deliveredAt` | Sí + `DELIVERED` |

Intentos fallidos reintentables: sin timestamp nuevo; macro **`FAILED`**; incluir `deliveryFailureContext` si el raw trae motivo/comentario (`comment` opcional).

**Otros carriers (futuro):** mismo patrón — mapear status → timestamp operativo; decidir emisión incremental vs gate terminal.

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
| 3b | OOLL entrega fallida reintentable | **Sí** | — (macro `FAILED`, sin milestone) | `deliveryFailureContext` |
| 3c | OOLL entrega fallida terminal | **Sí** | `FAILED` | subset mapeos (B.8.4) |
| 4 | OOLL entregada | **Sí** | `DELIVERED` | `deliveredAt` |
| 5 | Cancel / Andreani `Anulada` | **Sí** post-gate listable | `CANCELLED` | pre-gate: persist only |

**Watermark de proyección (B.2):** ver §5.1. Emisión = gate satisfecho ∧ delta vs watermark ∧ identidad proyectable.

### 5.1 Projection watermark (`package_projection_watermarks`)

**Schema API:** ✅ `PackageProjectionWatermark` → `package_projection_watermarks`.

**Implementación watermark:** diseño en este §; **status y archivos:** `plan.md` B.2.

**Propósito:** ledger de lo que API **reclamó** proyectar por bulto — no es el read model del projector.

| Campo | Rol |
|-------|-----|
| `lastMilestoneKey` | Último `MilestoneKey` incluido en una emisión |
| `projectedTimestamps` | JSON `{ readyToCollectAt: "…", oeLaunchAt: "…" }` — keys ya enviadas en `milestoneDeltas` / `operationalTimestamps` |
| `lastProjectedCanonicalStatus` / `lastProjectedSubStatus` | Último `stateChange` proyectado (OOLL sin avance de `MilestoneKey`) |
| `lastProjectedAt` | Timestamp del trigger de la última emisión reclamada |

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

- Delta multi-campo en bootstrap (`journeyAnnouncedAt` en chunk + `oeLaunchAt` en gate) y gates en orden no feliz.
- OOLL (B.5): emisiones incrementales con mismo `MilestoneKey`, cambios de `subStatus`, entrega fallida sin nuevo `*At`.

**Cohortes masivas:** watermark/outbox bulk en `publishManyEventsBulk` ✅; relay Fenix bulk secuencial → **`plan.md` § B.7** (B.7.1 ✅; B.7.2 pending).

---

## 6. Implementación vs definición

**Checklist y status:** única fuente → **`plan.md`**.

Desvíos abiertos (2026-06-24):

| Tema | Acción |
|------|--------|
| Fenix bulk relay API | B.7 — `plan.md` § B.7 |
| E2E local API → projector | B.6 — `local-testing-guide.md` |
| Cancel manual API | B.8.5 |
| `collectedAt` consumer primera milla | TBD |
| Catálogo milestones DB | C.2 persistencia (HTTP ✅) |
| Search index + list `q`/filtros | **DONE (C.5 + C.6)** · perf → `search-performance-spec.md` SP.1 |
| Tramo en `PackageEvent.shipmentId` | **C.4** — `blocks-projector.md` Block 6 |

El resto del camino warehouse + OOLL MVP v1 está alineado con este doc (ver evidencia en `plan.md` Ground Truth).

---

## 7. Referencias cruzadas

| Tema | Documento |
|------|-----------|
| Plan operativo + status | **`plan.md`** |
| Onboarding | `prompt.md` |
| E2E local | `local-testing-guide.md` |
| Fenix bulk relay | `plan.md` § B.7 |
| Terminales FAILED / CANCELLED | `plan.md` § B.8 |
| Blocks API | `blocks-api.md` |
| Blocks projector | `blocks-projector.md` |
| Flujo negocio | `logistics.md` |
