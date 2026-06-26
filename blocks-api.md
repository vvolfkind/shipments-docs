# Building Blocks: shipments-history-api — Event Emission for Projector

> **Status de implementación:** ver **`plan.md`** únicamente. Este archivo es **spec de diseño**; líneas `Status:` / DONE / PROPOSED dentro de cada block pueden estar desactualizadas.

## Overview

**Definiciones de dominio (milestones, tramos, estados, gates):** ver **`domain-model.md`** (autoridad). Este archivo detalla blocks de implementación API.

API reconcilia eventos de múltiples fuentes (inbox estandarizado de TMS, webhooks de carriers, etc.). Opera en **dos capas**:

1. **Persistencia operacional (API DB)** — evidencia cruda, identidad, timestamps operativos (`package_milestones`), tramos (`Shipment` + `PackageShipment`), timeline (`PackageEvent`). Ocurre tan pronto llega cada evento fuente (Block 2, 3, 6, 5…).
2. **Proyección analítica (projector)** — snapshots emitidos al topic cuando se cumple un **projection gate** (Block 1.11). Modo **OLAP**, no realtime por paso intermedio.

Emisiones al topic `shipments-history-api-to-projector`:
- **`PACKAGE_NEW_MILESTONE`** — para que projector consuma y proyecte (read model). Incluye `milestoneKey` + `operationalTimestamps` empaquetados (ver `domain-model.md` §4).
- **`JOURNEY_RECONCILIATION`** — señal interna de loop LPC (API roundtrip; projector omite).

**Camino warehouse-first (B.1):** Block 3 persiste identidad canónica y timestamps de primera milla en API; el **primer** contacto con projector ocurre en **`WM2_RS_MAIN_OE_LAUNCH`** (Block 6.3), gate **`WAREHOUSE_RECEIVED`**, empaquetando evidencia de primera milla (`readyToCollectAt`, `journeyAnnouncedAt`, `oeLaunchAt`, transición de tramos). Caminos sin warehouse usan **otros gates** — Block 5 / Block 8.

---

## Block 1: Emitir Evento Normalizado al Topic Fenix


**Objetivo:**
Reconciliar eventos de entrada (estado del carrier + contexto logístico) en agregado Package coherente, traducir estados a canonical, y emitir evento con payload completo para que projector proyecte localmente sin depender de consultas síncronas a API.

**Responsabilidades de API en Block 1:**

### 1.1 Inbox Processing
API recibe eventos de múltiples fuentes (TMS adapter, Enviopack webhooks, carrier status updates, WMS events). Cada fuente entra con identidad de carrier + raw payload.

**Entrada mínima esperada por fuente:**
- `trackingNumber` o similar (identificador del carrier)
- `status` raw del carrier (ej. "En tránsito", "SHIPPED", "transit_in_progress")
- `occurredAt` (Event Time del evento real)
- Metadatos operativos (ubicación, actor, etc.)

### 1.2 Diccionario de Traducción de Estados (API-side)
API mantiene tabla/config que mapea:
```
carrierCode + rawStatus → canonicalStatus + subStatus
```

**Ejemplo:**
```
ANDREANI + "En tránsito" → canonicalStatus: "IN_TRANSIT", subStatus: "TRANSPORT"
ENVIOPACK + "SHIPPED" → canonicalStatus: "IN_TRANSIT", subStatus: "DISPATCH"
FRAVEGA_WMS + "CLASSIFIED" → canonicalStatus: "WAREHOUSE_PROCESSING", subStatus: "CLASSIFIED"
```

**Responsabilidad:** API ejecuta traducción durante reconciliación. Evento que sale a Fenix ya contiene canonicalStatus normalizado.

### 1.3 Reconciliación de Package Identity

API debe resolver:
- Qué `externalId` usar (EP012933329N-01, tracking carrier, lastMileId, etc.)
- Qué `sourceSystem` asignar (ENVIOPACK, ANDREANI, FRAVEGA_WMS, etc.)
- Qué `orderId` vincular (si existe en datos comerciales)
- Qué `currentCarrierCode` y `currentTrackingId` (puede cambiar en el ciclo)

**Regla:** Una vez un package existe, API puede enriquecer con orden + delivery si no estaban antes. Pero externalId y sourceSystem no cambian (identidad inmutable).

### 1.4 Timestamps operativos vs milestones de proyección

> **Vocabulario:** Los campos `*At` en `PackageMilestones` son **timestamps operativos** (capa A en `domain-model.md`). Los **`MilestoneKey`** (5 valores v1: progreso + `DELIVERED` / `FAILED` / `CANCELLED`) son **milestones de proyección** (capa D). No son 1:1.

**Timestamps que API debe detectar y persistir (set-once salvo policy Block 7):**
- `readyToCollectAt`: listo para recoger (Block 2)
- `collectedAt`: recolección confirmada
- `journeyAnnouncedAt`: viaje LPC anunciado (Block 3 chunk)
- `oeLaunchAt`: llegada física al CD (Block 6.3)
- `classifiedAt`: clasificación a contenedor (Block 6.4 — **persist only**)
- `dispatchedAt`: salida física del CD (Block 6.5)
- `pickedUpByCarrierAt`, `consolidatedAtCarrierAt`, `inTransitAt`, `deliveredAt`: OOLL (Block 5)

**A) Persistencia (API DB):** cada fuente setea timestamps cuando reconcilia, **sin** esperar projection gate.

**B) Emisión (projector):** solo cuando se cumple un **projection gate** (Block 1.11). El payload incluye:
- `milestoneKey` — cuando el gate avanza catálogo UI (5 valores v1)
- `operationalTimestamps` — bundle de timestamps consolidados en ese gate
- `stateChange` — macro-estado (`IN_TRANSIT` / `DELIVERED`) + `subStatus` diagnóstico

**Alias legacy en código:** `milestoneDeltas` = `operationalTimestamps` hasta refactor de contrato. Semántica = timestamps empaquetados en gate, **no** un evento projector por cada campo.

**Clasificación WMS:** persiste `classifiedAt` pero **no** emite al projector; se empaqueta en gate `LAST_MILE_DISPATCHED`.

### 1.5 Shipment Linkage (tramos)

Modelo binario v1: `FIRST_MILE` | `LAST_MILE` (ver `domain-model.md` §3). `WAREHOUSE_INTERNAL` **no** se usa en v1.

Si el package está asociado a un tramo activo, API emite link:
```json
{
  "shipmentLink": {
    "shipmentId": "uuid-api",
    "trailType": "FIRST_MILE|LAST_MILE",
    "carrierCode": "ENVIOPACK|ANDREANI|...",
    "trackingNumber": "..."
  },
  "trailTransitions": {
    "closeFirstMile": { "exitedAt": "..." },
    "openLastMile": { "enteredAt": "...", "carrierCode": "...", "trackingNumber": "..." }
  }
}
```

**Reglas:**
- `FIRST_MILE` se crea en Block 2; se cierra en gate `OE_LAUNCH`.
- `LAST_MILE` se abre en gate `OE_LAUNCH` (incluye operativa CD + carrier); **no** en Block 3 chunk.
- Block 3 solo persiste `PackageIdentifierLink` tipo `LAST_MILE` cuando conoce tracking futuro.

### 1.6 Delivery Linkage

Si se conoce delivery comercial asociado:
```json
{
  "deliveryLink": {
    "deliveryId": "uuid-or-null"
  }
}
```

**Nota v1:** `deliveryLink` es opcional. Se incluye cuando vínculo comercial ya existe; se omite cuando aún no está reconciliado.

### 1.7 Evento Versionado y Determinístico

**Ejemplo A — bootstrap warehouse (gate `WAREHOUSE_RECEIVED`, trigger `OE_LAUNCH`):**

```json
{
  "eventVersion": 1,
  "eventType": "PACKAGE_NEW_MILESTONE",
  "externalEventId": "api-pkg-evt-{deterministic-hash}",
  "occurredAtUtc": "2026-06-08T19:37:22.484Z",
  "occurredAtRaw": "2026-06-08T19:37:22.484Z",
  "source": "SHIPMENTS_HISTORY_API",
  "milestoneKey": "WAREHOUSE_RECEIVED",

  "package": {
    "externalId": "EP012933329N-01",
    "sourceSystem": "ENVIOPACK",
    "orderId": "128-18733100-12991123",
    "currentCarrierCode": "ANDREANI",
    "currentTrackingId": "360002990645690"
  },

  "stateChange": {
    "status": "En tránsito",
    "subStatus": null,
    "canonicalStatus": "IN_TRANSIT",
    "lastOccurredAt": "2026-06-08T19:37:22.484Z"
  },

  "milestoneDeltas": {
    "readyToCollectAt": "2026-06-10T13:33:00.000Z",
    "journeyAnnouncedAt": "2026-06-11T10:30:00.000Z",
    "oeLaunchAt": "2026-06-08T19:37:22.484Z"
  },

  "trailTransitions": {
    "closeFirstMile": { "exitedAt": "2026-06-08T19:37:22.484Z" },
    "openLastMile": {
      "enteredAt": "2026-06-08T19:37:22.484Z",
      "carrierCode": "ANDREANI",
      "trackingNumber": "360002990645690"
    }
  },

  "shipmentLink": {
    "shipmentId": "uuid-of-shipment",
    "trailType": "LAST_MILE",
    "carrierCode": "ANDREANI",
    "trackingNumber": "360002990645690"
  },

  "deliveryLink": {
    "deliveryId": "uuid-or-null"
  },

  "metadata": {
    "emissionTrigger": "OE_LAUNCH",
    "journeyId": "uuid-journey",
    "entryOrderExternalId": "c90c5d7b-8311-4c80-b2aa-b945dff4a10d",
    "locationExternalId": "htyftyrAEdfGwQVwbhURrJ",
    "projectionBootstrap": true,
    "itemType": "HD"
  }
}
```

**Ejemplo B — evento incremental posterior (ej. OOLL Block 5):**

```json
{
  "eventVersion": 1,
  "eventType": "PACKAGE_NEW_MILESTONE",
  "externalEventId": "api-pkg-evt-{deterministic-hash}",
  "occurredAtUtc": "2026-06-17T14:25:00.000Z",
  "occurredAtRaw": "2026-06-17T11:25:00.000-03:00",
  "source": "SHIPMENTS_HISTORY_API",

  "package": {
    "externalId": "EP012933329N-01",
    "sourceSystem": "ENVIOPACK",
    "orderId": "128-18733100-12991123",
    "currentCarrierCode": "ANDREANI",
    "currentTrackingId": "360002990645690"
  },

  "stateChange": {
    "status": "IN_TRANSIT",
    "subStatus": "TRANSPORT",
    "canonicalStatus": "IN_TRANSIT",
    "lastOccurredAt": "2026-06-17T14:25:00.000Z"
  },

  "milestoneDeltas": {
    "inTransitAt": "2026-06-17T14:25:00.000Z"
  },

  "shipmentLink": {
    "shipmentId": "uuid-of-shipment",
    "trailType": "LAST_MILE",
    "carrierCode": "ANDREANI",
    "trackingNumber": "360002990645690"
  },

  "metadata": {
    "emissionTrigger": "OOLL_STATUS"
  }
}
```

**Timestamps del envelope vs milestones:**
- `occurredAtUtc` / `occurredAtRaw` = timestamp del **evento trigger** que abrió el projection gate (ej. `oeLaunchAt` en bootstrap warehouse).
- Cada key en `milestoneDeltas` conserva **su** timestamp histórico (puede ser anterior a `occurredAtUtc`).
- Projector ordena timeline por cada milestone, no solo por el envelope.

**externalEventId determinístico:**
- Hash de: `(source, package.externalId, emissionTrigger, triggerExternalEventId)`
- Ej. bootstrap warehouse: `(SHIPMENTS_HISTORY_API, EP012933329N-01, OE_LAUNCH, {entryOrderExternalId})`
- Garantiza idempotencia por bulto + trigger de cohorte
- Projector deduplica por `(source, externalEventId)`

**Regla de emisión v1:**
- API emite al topic cuando hay **delta de proyección** (watermark) **y** se cumple projection gate.
- Si hay cambio de status/subStatus sin milestone proyectable nuevo, no se emite.
- Bootstrap warehouse incluye snapshot de identidad/links ya persistidos en API (Block 3) aunque no sean milestones.

### 1.8 Chunking para Picos 8x

Si payload > 512 KB soft cap:
- API particiona eventos
- Todos comparten reconciliationId determinístico
- Emite múltiples mensajes con chunkIndex / chunkTotal
- Fenix mantiene order per partition key = `package.externalId`

**Responsabilidad:** API calcula soft cap y parte si necesario. No supera hard cap 1 MB menos headers.

### 1.9 Fenix Topic Destination

**Topic:** `shipments-history-api-to-projector`  
**Env API:** `NOTIFICATIONS_SYNC_API_TO_PROJECTOR`  
**Env projector:** `FENIX_QUEUES_SYNC_API_TO_PROJECTOR`

Un **mismo topic** transporta señales internas de reconciliación LPC y eventos de proyección. Se discriminan por `eventType` (y por `eventName` en el envelope Fenix).

| `eventType` | Consumidor | Propósito |
|---|---|---|
| `JOURNEY_RECONCILIATION` | **API** (roundtrip Fenix → `/ingestor/fenix-queue/consume`) | Loop de reconciliación por chunks; projector **omite** |
| `PACKAGE_NEW_MILESTONE` | **Projector** | Proyección read model; API no re-consume |

**Partition key (`PACKAGE_NEW_MILESTONE`):** `package.externalId` (garantiza orden por bulto)

**Partition key (`JOURNEY_RECONCILIATION`):** `journeyId` (recomendado; orden por viaje LPC)

**Retention:** 7 días mínimo (permite replay, backfill puntual)

### 1.10 Señal interna — `JOURNEY_RECONCILIATION`

Señal mínima para el loop de reconciliación Block 3.2. **No** es evento de dominio para projector.

```json
{
  "eventVersion": 1,
  "eventType": "JOURNEY_RECONCILIATION",
  "eventName": "JOURNEY_RECONCILIATION",
  "source": "SHIPMENTS_HISTORY_API",
  "data": {
    "journeyId": "uuid",
    "limit": 500
  }
}
```

- `limit` opcional; default = `Journey.chunkSize` o `app_config`.
- Fenix `queue` = `shipments-history-api-to-projector`; `eventName` = `JOURNEY_RECONCILIATION`.
- API debe estar suscripta a ese topic para recibir el roundtrip (infra Fenix).
- Projector registra handler no-op o filtra por `eventType` y **no procesa** esta señal.

### 1.11 Projection gates (cuándo emitir al projector)

API persiste timestamps operativos antes de emitir. La emisión obedece **gates** — ver catálogo completo en `domain-model.md` §4–§5.

| Gate | Trigger | `MilestoneKey` | Emite | Notas |
|------|---------|----------------|-------|-------|
| 1 | `WM2_RS_MAIN_OE_LAUNCH` | `WAREHOUSE_RECEIVED` | Sí (bootstrap) | Primer contacto projector; cierra `FIRST_MILE`, abre `LAST_MILE` |
| — | `WM2_PD_MAIN_CLASSIFICATION` | — | **No** | Persist `classifiedAt` + metadata |
| 2 | `CONTAINER_PACKAGES_SHIPPED` | `LAST_MILE_DISPATCHED` | Sí | Incluye `classifiedAt`, LPN, operario |
| 3 | OOLL checkpoints (Block 5) | — | Sí (incremental) | Actualiza timestamps/estado; **no** avanza `MilestoneKey` |
| 4 | OOLL entregada | `DELIVERED` | Sí | Terminal v1 |
| — | Block 2 / Block 3 chunk | — | **No** | Persist only |

**Watermark de proyección (B.2):** tabla `package_projection_watermarks` (schema ✅). **B.2.2 DONE:** repo + `PackageProjectionDeltaService`. **B.2.3–B.2.4 pending:** lectura en gate use case; claim (`claimBulk`) en `FenixOutboxProducer` tx con outbox `PENDING` — activado **post-return** por `EventPublisherInterceptor`. Detalle: `domain-model.md` §5.1.

**Cohorte warehouse:** `OE_LAUNCH` por `entryOrderExternalId` = `Journey.externalId`. Emisión per-package; `metadata.projectionBootstrap: true` en gate 1. Replay del mismo gate: `appliedPackageIds` + set-once (`oe_launch_at` / `dispatched_at`) evita re-emisión sin depender solo del watermark.

**Out-of-order:** Block 6.6 — persist válido; emitir cuando identidad canónica + gate + delta vs watermark.

### 1.12 Read model y frontend (política OLAP)

- Grilla: `current_milestone` = `MilestoneKey` (5 valores v1), **no** `subStatus`.
- Detalle: timeline + timestamps operativos + `segments[]` por `trailType`.
- Macro-estado visible: `IN_TRANSIT` hasta `DELIVERED`.
- Gate `WAREHOUSE_RECEIVED` empaqueta timestamps de primera milla (`readyToCollectAt`, `journeyAnnouncedAt`, `oeLaunchAt`) — primera vez visibles en projector.
- Operativa CD visible en detalle vía timestamps/metadata; milestone UI avanza en `LAST_MILE_DISPATCHED`.

### 1.13 Camino non-warehouse (futuro, fuera B.1)

Bulto con hitos OOLL (ej. `inTransitAt`) **sin** `oeLaunchAt` / sin `JourneyPackage` → señal analítica de **non-warehouse path** (nunca pasó por CD en nuestro modelo). Reconciliación y projection gate en Block 5 / Block 8 — **no** reutilizar bootstrap warehouse de dos hitos.

---

## Block 2: Initial Package Registration from ep-ready-to-collect-orders


**Objetivo:**
Cuando EnvioPack informa que un bulto está listo en su depósito, crear Package canónico e inmediatamente crear Shipment de FIRST_MILE asociado. Este es el primer milestone observable.

**Nota de estrategia (B.1 primero):**
- Este block crea entidad en estado borrador operativo cuando solo existe `EP...N` sin secuencia.
- El camino prioritario de reconciliación es WMS/LPC (warehouse-first).
- Reconciliación alternativa no-LPC se define en Block 8 (B.2).
- `DRAFT_BASE` es estado interno de API; no forma parte del contrato proyectable hacia projector.

**Entrada:**
```json
{
  "trackingNumber": "EP012777046N",
  "receivedDate": "2026-06-10T13:33:38.872Z"
}
```

**Procesamiento Block 2:**

### 2.1 Create or Upsert Package
```
Package {
  externalId: "EP012777046N",
  sourceSystem: "ENVIOPACK",
  currentCarrierCode: "ENVIOPACK",
  currentTrackingId: "EP012777046N",
  status: "READY_TO_COLLECT",
  subStatus: "READY_AT_FIRST_MILE",
  lastOccurredAt: "2026-06-10T13:33:38.872Z",
  metadata: {
    "ep_base": "EP012777046N",
    "ep_raw_event": {...},
    "has_sequence": false,  // aún no sabemos -01, -02, etc
    "identity_state": "DRAFT_BASE"
  }
}
```

**Regla importante:** externalId = base sin secuencia (EP012777046N). Cuando llegue LPC con múltiples bultos, crearemos packages con EP012777046N-01, EP012777046N-02, etc., pero todos compartirán un `groupId` o `parentExternalId` para vinculación.

**Regla adicional v1:**
- `EP...N` se considera `draft package` hasta reconciliación por LPC.
- Si no hay LPC, queda pendiente para reconciliación alternativa (Block 8).
- Mientras siga en `DRAFT_BASE`, API no debe emitirlo al projector como entidad visible.
- Projector solo recibe identidades concretas/canónicas o suficientemente estabilizadas para proyección.

**Nota de evolución:**
- Si negocio necesita visibilidad más temprana que `ep-ready-to-collect-orders`, esa precisión debe venir por evolución de API (nuevos consumers/orígenes descubiertos aguas arriba de EnvioPack/OOLL), no trasladando drafts al read model.

### 2.2 Create Shipment FIRST_MILE
```
Shipment {
  sourceSystem: "ENVIOPACK",
  trailType: "FIRST_MILE",
  carrierCode: "ENVIOPACK",
  trackingNumber: "EP012777046N",
  status: "CREATED",
  canonicalStatus: "CREATED"
}
```

### 2.3 Link Package to Shipment
```
PackageShipment {
  packageId: <package_id>,
  shipmentId: <shipment_id>,
  enteredAt: "2026-06-10T13:33:38.872Z"
}
```

### 2.4 Create Initial PackageMilestones Row
```
PackageMilestones {
  packageId: <package_id>,
  readyToCollectAt: "2026-06-10T13:33:38.872Z"
}
```

### 2.5 Create PackageEvent (append-only)
```
PackageEvent {
  packageId: <package_id>,
  shipmentId: <shipment_id>,
  source: "ENVIOPACK",
  topic: "ep-ready-to-collect-orders",
  eventName: "EP_READY",
  status: "READY_TO_COLLECT",
  canonicalStatus: "CREATED",
  occurredAt: "2026-06-10T13:33:38.872Z",
  externalEventId: "ep-ready-{hash(trackingNumber, receivedDate)}"
}
```

**Regla de idempotencia:** (packageId, source, externalEventId) UNIQUE → rechaza duplicados sin error.

**Qué falta en Block 2 (vs definición `domain-model.md`):**
- ~~Crear `Shipment FIRST_MILE` + `PackageShipment`~~ **DONE** (`ShipmentRepository`, ep-ready use case)
- Idempotencia EP tracking duplicado
- Proyección: **no** en Block 2; primer gate = `OE_LAUNCH`

**Próximo paso:** Block 3 — WMI anuncia viaje LPC; loop Fenix reconcilia `EP...N` draft → `EP...N-0x` canónico (persistencia API; proyección en Block 6.3).

---

## Block 3: WMI Journey Announcement + LPC Reconciliation (Fenix Loop)


**Objetivo:**
Cuando llega el anuncio WMI de viaje LPC, materializar el viaje operativo (`Journey`) y reconciliar sus `receiptItems` (1..N bultos, hasta decenas de miles) **sin** procesar todo en memoria ni bloquear el consumer HTTP.

**Separación semántica obligatoria:**
- **`Journey`** = viaje operativo de colecta LPC (muchos bultos, un camión/ruta hacia warehouse). **No** es última milla.
- **`Shipment`** = tramo logístico con operador (`FIRST_MILE`, `LAST_MILE`). Ver `domain-model.md` §3.
- **Block 3 chunk:** persiste identifiers `LAST_MILE` cuando WMI trae `lastMileId`; **no** crea `Shipment LAST_MILE` (se abre en gate `OE_LAUNCH`).

**Modalidades `receiptItem` (WMI):**
- **`itemType: HD`** — `destination.warehouseId = null`, `destination.OOLL.courierName` obligatorio. Scope v1 Andreani/última milla vía `lastMileId`.
- **`itemType: SPU`** — `destination.warehouseId` obligatorio (sucursal pickup; **no** es warehouse CD), `destination.OOLL = null`. **No se excluye** del pipeline: se persisten, reconcilian y enriquecen milestones hasta donde el contrato actual lo permite. Gaps de discovery quedan explícitos en metadata para analistas hasta integrar Unigis u otros orígenes de recepción en sucursal.

**Resultado esperado de canonización LPC (B.1):**
- `EP...N` borrador (Block 2) se reconcilia con `EP...N-01..N-k` del WMI (1 draft → N canónicos posibles; distinto `lastMileId` por bulto HD).
- Cada bulto canónico recibe `journeyAnnouncedAt` en API en este block (chunk).
- `oeLaunchAt` **no** se setea acá — llega en `WM2_RS_MAIN_OE_LAUNCH` (Block 6.3).
- **No** emitir `PACKAGE_NEW_MILESTONE` desde chunk: proyección diferida al gate OE_LAUNCH (Block 1.11).

**Entrada WMI (notification + fetch `WMI_API_WEBHOOK_URL/{wmiExternalId}`):**
```json
{
  "pickupType": "sellers",
  "warehouseId": "180",
  "appointment": {
    "expectedDate": "2025-06-10T13:22:58.078Z",
    "journeyId": "1234",
    "route": "#0004"
  },
  "receiptItems": [
    {
      "id": "EP007381498N-01",
      "deliveryId": "128-1234567-123456",
      "lastMileId": "360002990645690",
      "group": { "id": "EP007381498N", "quantity": 1 },
      "destination": { "warehouseId": "03", "OOLL": { "courierName": "Andreani" } }
    }
  ]
}
```

---

### 3.0 Schema API — entidades nuevas

**Ubicación:** `shipments-history-api/src/infrastructure/database/schema.prisma`  
**Tablas:** `journeys`, `journey_receipt_items` (staging), `journey_packages`  
**Alias histórico:** `PickupJourney` / `PickupJourneyPackage` en `logistics.md` → `Journey` / `JourneyReceiptItem` + `JourneyPackage` acá.

Agregar al schema de `shipments-history-api` (ya modelado en Prisma; migración la corre el operador):

#### A) `journeys` — cabecera del viaje LPC

```
Journey {
  id: UUID PK
  sourceSystem: "WMI"                    // origen del anuncio
  externalId: VARCHAR(128)               // wmiExternalId / journeyUuid
  journeyType: "LPC"
  status: PENDING_RECONCILIATION | RECONCILING | RECONCILED | FAILED
  announcedAt: TIMESTAMPTZ               // timestamp del anuncio WMI
  warehouseId: VARCHAR(32) | null
  routeCode: VARCHAR(64) | null
  totalReceiptItems: INT                // count(receiptItems) al persistir staging
  processedReceiptItems: INT DEFAULT 0  // avance reconciliación
  lastProcessedOffset: INT DEFAULT 0    // legacy/resumen; el loop usa claim por status, no offset
  chunkSize: INT DEFAULT 500            // tamaño de lote (configurable: 100, 200, 500, app_config)
  rawPayload: JSONB                     // snapshot WMI completo (opcional si staging guarda items)
  metadata: JSONB
  createdAt, updatedAt

  UNIQUE(sourceSystem, externalId)
  INDEX(status, updatedAt)
}
```

#### B) `journey_receipt_items` — staging consultable (recomendado para 1..N alto)

Tabla de staging **desacoplada** del upsert de `packages`. El loop reclama lotes por status + lock, sin re-parsear JSON ni retener 20k items en memoria.

```
JourneyReceiptItem {
  id: UUID PK
  journeyId: UUID FK -> journeys
  lineIndex: INT                         // orden estable 0..N-1
  packageExternalId: VARCHAR(256)        // receiptItems[].id (EP...N-01)
  deliveryId: VARCHAR(64) | null
  lastMileId: VARCHAR(128) | null
  groupId: VARCHAR(128) | null           // receiptItems[].group.id (EP...N base)
  courierName: VARCHAR(64) | null        // HD: destination.OOLL.courierName; SPU: null
  rawItem: JSONB                         // receiptItem completo (incl. itemType, destination)
  reconciliationStatus: PENDING | PROCESSING | PROCESSED | FAILED
  lockedAt: TIMESTAMPTZ | null           // claim horizontal (K8s)
  lockedBy: VARCHAR(253) | null          // pod / consumer id
  packageId: UUID | null                 // FK packages, set al procesar chunk
  processedAt: TIMESTAMPTZ | null
  lastError: TEXT | null
  createdAt, updatedAt

  UNIQUE(journeyId, lineIndex)
  UNIQUE(journeyId, packageExternalId)
  INDEX(journeyId, lineIndex)
  INDEX(journeyId, reconciliationStatus)
}
```

#### C) `journey_packages` — vínculo operativo package ↔ journey (post-upsert)

```
JourneyPackage {
  journeyId: UUID FK
  packageId: UUID FK
  role: VARCHAR(32) | null
  enteredAt: TIMESTAMPTZ                 // announcedAt del journey
  metadata: JSONB
  createdAt

  PRIMARY KEY (journeyId, packageId)
  INDEX(packageId)
}
```

**Nota:** `Journey` **no** reemplaza `Shipment`. Última milla sigue modelada con `Shipment.trailType = LAST_MILE` + `PackageShipment`.

---

### 3.1 Fase síncrona — `ConsumeWmiJourneyAnnouncementUseCase`

**Responsabilidad:** terminar en **"journey persisted, reconciliation loop started"**. No reconciliar 20k bultos inline.

**Flujo v1:**

1. Persistir `InboxEvent` (implementado).
2. Fetch WMI `GET WMI_API_WEBHOOK_URL/{wmiExternalId}` (implementado).
3. Upsert `Journey` por `(sourceSystem="WMI", externalId=wmiExternalId)`:
   - `status = PENDING_RECONCILIATION`
   - `announcedAt` = `eventTimestamp` del notification (fallback `now()`)
   - `totalReceiptItems = receiptItems.length`
   - `rawPayload` = snapshot completo
4. Bulk insert idempotente en `journey_receipt_items`:
   - una fila por `receiptItems[i]` con `lineIndex = i` (**HD y SPU**)
   - `reconciliationStatus = PENDING`
   - `createMany({ skipDuplicates: true })` o `ON CONFLICT DO NOTHING`
5. **Emitir señal Fenix** `JOURNEY_RECONCILIATION` al topic `shipments-history-api-to-projector` con `{ journeyId }`. No esperar fin del loop en HTTP handler.
6. Responder:

```json
{
  "journeyId": "uuid",
  "wmiExternalId": "c90c5d7b-8311-4c80-b2aa-b945dff4a10d",
  "receiptItemCount": 20000,
  "reconciliationStatus": "QUEUED"
}
```

**Límites explícitos de la fase síncrona (Block 6.2):**
- **Sí:** persistir journey + staging + emitir primera señal `JOURNEY_RECONCILIATION`.
- **No:** upsert masivo de `packages` inline.
- **No:** setear `journeyAnnouncedAt` inline (lo hace el chunk use case).
- **No:** setear `oeLaunchAt`.
- **No:** emitir `PACKAGE_NEW_MILESTONE` masivo.
- **No:** crear `Shipment LAST_MILE` inline (Block 3.3).

**Idempotencia:** re-procesar mismo `wmiExternalId`:
- `Journey` upsert no duplica.
- `journey_receipt_items` dedupe por `(journeyId, packageExternalId)`.
- re-emitir señal safe si quedaron items `PENDING`.

---

### 3.2 Loop de reconciliación — Fenix roundtrip (no worker/job)

**Nombre sugerido:** `ProcessJourneyReceiptChunkUseCase`

**Modelo:** no es worker Go ni cron. Es un **use case invocable** por:
1. **Roundtrip Fenix** — handler `JOURNEY_RECONCILIATION` en topic `shipments-history-api-to-projector` (prod).
2. **REST manual** — `POST /ingestor/journeys/:journeyId/reconcile/chunk` (dev/ops); debe usar el **mismo** use case (idealmente mismo produce → consume para paridad).

**Input:**

```typescript
{
  journeyId: string;   // UUID — PK de journeys
  limit?: number;      // default Journey.chunkSize o app_config (100 | 200 | 500)
}
```

**Paso 1 — claim de lote (inicio de cada invocación):**

```sql
UPDATE journey_receipt_items
SET reconciliation_status = 'PROCESSING',
    locked_at = now(),
    locked_by = :consumerId
WHERE id IN (
  SELECT id FROM journey_receipt_items
  WHERE journey_id = :journeyId
    AND reconciliation_status = 'PENDING'
  ORDER BY line_index ASC
  LIMIT :limit
  FOR UPDATE SKIP LOCKED
)
RETURNING *;
```

**Regla de parada del loop (crítica):**
- Si el claim devuelve **0 filas** → **no emitir** `JOURNEY_RECONCILIATION`; marcar `Journey.status = RECONCILED` si no quedan `PENDING`/`PROCESSING`; `return`.
- Si hay filas → procesar chunk → **siempre emitir** `JOURNEY_RECONCILIATION` al terminar (sin evaluar `remaining` antes de emitir).
- La invocación siguiente recibe la señal, ejecuta paso 1 vacío y cierra el loop. Costo: 1 roundtrip Fenix extra por journey.

**Output:**

```typescript
{
  journeyId: string;
  limit: number;
  claimedCount: number;
  processedCount: number;
  failedCount: number;
  continued: boolean;           // true si emitió señal de continuación
  journeyStatus: "RECONCILING" | "RECONCILED" | "FAILED";
}
```

**Orquestación:**
- Primer chunk con filas: `Journey.status = RECONCILING`.
- Tras chunk: incrementar `processedReceiptItems`; filas → `PROCESSED` o `FAILED`.
- Locks stale en `PROCESSING` (pod crash): recuperación por timeout + retry (operacional).
- Escala horizontal: `SKIP LOCKED` evita que dos pods reclamen la misma fila.

**Señal de continuación (siempre post-chunk con trabajo):**

```json
{
  "eventType": "JOURNEY_RECONCILIATION",
  "journeyId": "uuid",
  "limit": 500
}
```

Topic: `shipments-history-api-to-projector` (`NOTIFICATIONS_SYNC_API_TO_PROJECTOR`).  
**No** reutilizar la queue WMI (`WMI_ENTRY_ORDER_NOTIFY`); es roundtrip sobre el topic sync API↔projector.

**Restricción memoria/CPU (K8s):**
- Nunca cargar `receiptItems` completo del WMI en memoria.
- Solo el lote claimado (`limit` rows) + lookups batch (`WHERE external_id IN (...)`).
- Transacción **por chunk**, no por journey completo.

---

### 3.3 Qué hace el chunk por cada `receiptItem`

Por cada fila claimada (`PROCESSING` → `PROCESSED` / `FAILED`):

**Común (HD + SPU):**

1. **Upsert `Package` canónico** por `packageExternalId` (`EP...N-01`):
   - si no existe → CREATE con `sourceSystem = ENVIOPACK`
   - si existía draft `EP...N` (Block 2) → reconciliar metadata (`ep_base`, `ep_sequence`, `group_id`)
   - enriquecer con `deliveryId`, metadata de `rawItem`
2. **Upsert `PackageIdentifierLink`:**
   - `EP_FULL` → `packageExternalId`
   - `EP_BASE` → `group.id` cuando exista
   - `SHIPPING_ID` / equivalente → `deliveryId` si aplica
3. **Upsert `PackageMilestones`:**
   - setear **`journeyAnnouncedAt` only** (`set-once`, timestamp = `Journey.announcedAt`)
   - **no** tocar `oeLaunchAt`, `classifiedAt`, `dispatchedAt`, ni milestones OOLL
4. **Insert `JourneyPackage`** idempotente.
5. **Append `PackageEvent` técnico** (opcional v1): `JOURNEY_ANNOUNCED`, dedupe por `(journeyId, packageExternalId)`.

**Rama HD (`itemType: HD`):**

6. **Upsert `PackageIdentifierLink`:** `LAST_MILE` → `lastMileId` (discovery temprano; **no** crear `Shipment` aún).
7. **No** crear `Shipment LAST_MILE` en chunk — se abre en gate `OE_LAUNCH` (Block 6.3).
8. Andreani timeline posterior vía `andreani-events` (Block 5).

**Rama SPU (`itemType: SPU`):**

6. Persistir `pickupStoreId` = `destination.warehouseId` en metadata del package (sucursal pickup, no CD).
7. **No** crear `Shipment LAST_MILE` (aún no hay OOLL en contrato WMI).
8. Exponer **discovery gaps** en metadata, ej. `discoveryGaps: ["STORE_RECEIPT", "LAST_MILE_TRACKING"]`, para visibilidad analítica hasta integrar Unigis u otros feeds.

**Reconciliación draft → canónico:**
- Match por `group.id` (EP base) entre draft Block 2 y `EP...N-0x` del WMI.
- Un draft `EP...N` puede expandirse a **N** packages canónicos; draft queda como evidencia API (superseded), no identidad proyectable.
- Draft `EP...N` no se emite a projector; identidad proyectable = `EP...N-0x` vía gate OE_LAUNCH.

---

### 3.4 Proyección al projector desde el chunk — **no emite**

Block 3 chunk **persiste only** en API:
- identidad canónica, links (`LAST_MILE` identifier only — **no** `Shipment`), `journeyAnnouncedAt`, `JourneyPackage`, metadata SPU.

**No** crear `OutboxEvent` / `PACKAGE_NEW_MILESTONE` desde el chunk ni desde WMI sync HTTP.

La primera emisión al projector ocurre en **Block 6.3** (`OE_LAUNCH`), gate `WAREHOUSE_RECEIVED` con bundle de timestamps primera milla + transición de tramos (`domain-model.md` §4.2).

Metadata SPU (`itemType`, `pickupStoreId`, `discoveryGaps`) viaja en el **bootstrap** del primer evento projector, no antes.

---

### 3.5 Timestamps, tramos y emisión — regla Block 6.2 (cerrada)

| Hito / acción | WMI sync | Chunk | OE_LAUNCH |
|---|---|---|---|
| `journeyAnnouncedAt` | No | **Sí** persist | Bundle gate 1 |
| `oeLaunchAt` | No | No | **Sí** persist + gate |
| `Shipment FIRST_MILE` | No | No | **Cerrar** (`exitedAt`) |
| `Shipment LAST_MILE` | No | **No crear** | **Abrir** (`enteredAt`) |
| `PACKAGE_NEW_MILESTONE` | No | No | **Sí** (`WAREHOUSE_RECEIVED`) |
| `JOURNEY_RECONCILIATION` | Sí (primera) | Sí (continuación) | No |

**Prohibido en Block 3 / WMI path:**
- `PACKAGE_NEW_MILESTONE` masivo desde consumer HTTP
- setear `oeLaunchAt` antes de `WM2_RS_MAIN_OE_LAUNCH`
- bloquear request WMI hasta procesar N completo
- excluir items SPU del staging o de la reconciliación

---

### 3.6 Criterios de validación Block 3

- WMI consumer con 20k `receiptItems` responde rápido: journey + staging + primera señal Fenix.
- Loop procesa chunks claimados sin OOM; re-run idempotente con locks.
- Cada package (HD y SPU) del journey tiene `journeyAnnouncedAt` y `JourneyPackage` **en API**.
- HD: `lastMileId` en identifier links; **no** `Shipment LAST_MILE` hasta OE_LAUNCH.
- SPU: `pickupStoreId` en metadata; gaps visibles; sin LAST_MILE prematuro.
- Re-procesar mismo WMI no duplica packages/events/links.
- Chunk **no** emite `PACKAGE_NEW_MILESTONE`; `JOURNEY_RECONCILIATION` ignorado por projector.

**Regla de frontera API → Projector (cierra B.1):**
- API no emite `PACKAGE_NEW_MILESTONE` para `EP...N` mientras siga siendo `DRAFT_BASE`.
- Primer emisión de canónico `EP...N-0x` = gate **OE_LAUNCH** con batch de dos milestones.
- v1 no requiere evento alias draft→canónico en projector.

**Próximo paso:** Block 4 (outbox + relay) + Block 6.3 (`ConsumeOeLaunchUseCase`) + consumer projector `PACKAGE_NEW_MILESTONE`.

---

## Block 4: Emit to Fenix Topic (`PACKAGE_NEW_MILESTONE` + relay Outbox)


**Objetivo:**
Persistir `OutboxEvent` y relay al topic `shipments-history-api-to-projector` cuando se cumple un **projection gate** (Block 1.11). Primer gate B.1 = **OE_LAUNCH**. El **mismo topic** transporta señales internas **`JOURNEY_RECONCILIATION`** (Block 1.10); el relay/outbox de milestones **no** confunde ambos tipos.

**Mecánica v1 (cerrada):** el gate use case **no** escribe outbox; devuelve `{ event }`. `EventPublisherInterceptor` (post-return) invoca `FenixOutboxProducer`, que en una tx reclama watermark + inserta outbox `PENDING`, luego relay Fenix. Ver §4.1 watermark y `domain-model.md` §5.1.

### 4.1 Triggers de emisión (per package)

Condición base: identidad proyectable (`EP...N-0x` canónico) + **delta de proyección** (watermark) + projection gate satisfecho.

**No dispara `PACKAGE_NEW_MILESTONE`:**
- persistencia sync WMI (`Journey` + staging only)
- chunk Block 3 (aunque setee `journeyAnnouncedAt` en API)
- enriquecimiento de links/metadata sin gate ni delta de proyección
- señales `JOURNEY_RECONCILIATION` (internas API)
- draft `DRAFT_BASE` (`EP...N` sin secuencia)

**Sí dispara `PACKAGE_NEW_MILESTONE` (por package):**

| Trigger | Gate | `MilestoneKey` | Bundle típico |
|---|---|---|---|
| **`WM2_RS_MAIN_OE_LAUNCH`** | 1 — bootstrap | `WAREHOUSE_RECEIVED` | Timestamps primera milla + `oeLaunchAt`; transición tramos |
| **`WM2_PD_MAIN_CLASSIFICATION`** | — | — | **No emite** — persist only |
| **`CONTAINER_PACKAGES_SHIPPED`** | 2 | `LAST_MILE_DISPATCHED` | `classifiedAt`, `dispatchedAt`, `warehouseContext` |
| **Andreani / OOLL** (Block 5) | 3 / 4 | — o `DELIVERED` | Timestamps OOLL; ver matriz 5.3 |

**Bootstrap warehouse (`metadata.projectionBootstrap: true`):**
- Upsert package en projector (identidad + links persistidos en Block 3).
- `milestoneKey: WAREHOUSE_RECEIVED`
- `stateChange.canonicalStatus: IN_TRANSIT` (macro-estado; ver `domain-model.md` §2)
- `metadata.emissionTrigger: "OE_LAUNCH"`, `journeyId`, `entryOrderExternalId`, `locationExternalId`.

```
OutboxEvent {
  aggregateType: "PACKAGE",
  aggregateId: <package_id>,
  eventType: "PACKAGE_NEW_MILESTONE",
  payload: {
    // Evento normalizado Block 1.7
    eventVersion: 1,
    externalEventId: "api-pkg-evt-{hash}",
    occurredAtUtc: <trigger event timestamp — ej. oeLaunchAt>,
    occurredAtRaw: <raw from WMS OE_LAUNCH>,
    source: "SHIPMENTS_HISTORY_API",

    package: { externalId, sourceSystem, orderId, currentCarrierCode, currentTrackingId },

    stateChange: { status, subStatus, canonicalStatus, lastOccurredAt },

    milestoneDeltas: {
      // bootstrap B.1: journeyAnnouncedAt + oeLaunchAt
      // incremental: solo keys con delta de proyección
    },

    shipmentLink: { ... },  // bootstrap: LAST_MILE si HD
    deliveryLink: { ... },

    metadata: {
      emissionTrigger: "OE_LAUNCH",
      projectionBootstrap: true,
      journeyId: <uuid>,
      entryOrderExternalId: <uuid>,
      locationExternalId: <string>,
      itemType: "HD|SPU",
      pickupStoreId: <SPU optional>,
      discoveryGaps: <SPU optional>
    }
  },

  status: "PENDING",
  createdAt: now()
}
```

**Watermark API (B.2 — diseño cerrado; B.2.2 coded):**

| Subtask | Ubicación | Status |
|---------|-----------|--------|
| B.2.2 Repo | `repositories/packageProjectionWatermark/` | **DONE** |
| B.2.2b Delta service | `application/packageProjectionWatermark/packageProjectionDelta.service.ts` | **DONE** |
| B.2.3 Gate integration | `consumeOeLaunch`, `consumeWmsContainerPackagesShipped`, `consumeAndreaniEvents` | **DONE** |
| B.2.4 Producer claim | `framework/.../fenix.outbox.producer.ts` | **DONE** |
| B.7 Fenix bulk relay | `fenix.outbox.producer.ts` → chunks secuenciales → `produceBulk` | **DONE (core)** — B.7.4 docs pending; E2E `warehouse` `2026-06-25T16-59-55-015Z` (`plan.md` § B.7) |

**Gate use case (pre-return):**
1. Persist operativo (como hoy).
2. **Leer** watermark (bulk) → calcular delta → `buildProjectionEvents()` solo con keys/`stateChange` no reclamados.
3. `return { event: EventResponse | EventResponse[] }` — **sin** insert outbox ni upsert watermark.

**`EventPublisherInterceptor` (post-return):** si `response.event` presente → `FenixOutboxProducer.publishManyEvents`.

**`FenixOutboxProducer` (activación outbox):**
1. `BEGIN TX` → bulk upsert `package_projection_watermarks` (claim desde payload) + `createManyAndReturn` `outbox_ingestor` `PENDING`.
2. `COMMIT` → relay Fenix: cohort LPC vía **`POST /api/event/bulk`** en chunks N secuenciales (**B.7.2** ✅); retries con `eventId` → `/api/event` single.
3. Reintentos con `event.payload.eventId` → relay sequential `/api/event` sin re-claim (path existente).

**Anti-dup:** warehouse — `appliedPackageIds` + set-once; multi-gate/OOLL — watermark; entrega — outbox; projector — `externalEventId` determinístico.

**No hacer:** upsert watermark u outbox desde el use case; actualizar watermark post-`SENT`; worker post-proceso dedicado al watermark.

### 4.2 Outbox Relay Consumer
Proceso separado que:
1. Lee OutboxEvent con status=PENDING y `eventType = PACKAGE_NEW_MILESTONE`
2. Emite a Fenix topic `shipments-history-api-to-projector`
3. Marca OutboxEvent como PROCESSED

**Regla:** Relay debe respetar partition key = `package.externalId` para garantizar orden causal por bulto.

**Projector:** consume solo `PACKAGE_NEW_MILESTONE`; ignora `JOURNEY_RECONCILIATION`.

**API (mismo topic):** consume `JOURNEY_RECONCILIATION` vía handler dedicado; no procesa sus propios `PACKAGE_NEW_MILESTONE` salvo que se defina relay inverso (no v1).

**Qué falta en Block 4:**
- Decidir frecuencia de batching (cada evento inmediatamente, o batch por tiempo/tamaño)
- Retry logic en relay (DLQ si Fenix no responde)
- Feature flag para apagar relay en desarrollo

**Próximo paso:** Projector consume evento del topic y proyecta.

---

## Block 5: Reconciliación Last Mile (OOLL) y Emisión


**Objetivo:**
Definir reconciliación específica de eventos de última milla (Andreani primero; otros operadores vía consumers dedicados), incluyendo reglas de milestones intermedios (`pickedUpByCarrierAt`, `consolidatedAtCarrierAt`, `inTransitAt`) y emisión consistente a outbox. Motor OOLL compartido: **§5.9** (`OollProcessingService`).

### 5.1 Fuente de verdad para traducción

La traducción `rawStatus(OOLL) -> status/subStatus` ya existe en el seeder de `CarrierStatusMapping`.

Regla v1:
- API no define milestones contra estados raw del operador.
- API resuelve primero el estado estandarizado.
- Projector recibe siempre `status` estandarizado.
- `subStatus` queda como granularidad operativa y puede ser `null`.

### 5.2 Clave operativa para resolver milestones

Aunque el diccionario estandariza, `status` por sí solo no alcanza para decidir milestones de OOLL.

Ejemplo:
- `En tránsito` puede significar `Recepcionado por OOLL`.
- `En tránsito` puede significar `En sucursal de distribución`.
- `En tránsito` puede significar `En distribución`.
- `En tránsito` puede significar `Entrega fallida`.

Regla v1:
- la matriz se evalúa sobre el par `(status, subStatus)`.
- **Todo evento OOLL con cambio de estado y/o timestamp operativo debe proyectar** al topic (`domain-model.md` §4.2 gate 3+).
- si el par no mapea timestamp pero cambia `subStatus` (ej. `Entrega fallida`), igualmente emite con `stateChange` (+ `deliveryFailureContext` si aplica).

### 5.3 Matriz operativa v1 (`status` + `subStatus` -> `milestoneDelta`)

| status | subStatus | milestoneDelta | writePolicy | Emisión |
|---|---|---|---|---|
| `En tránsito` | `Recepcionado por OOLL` | `pickedUpByCarrierAt` | `set-once` | sí |
| `En tránsito` | `En sucursal de distribución` | `consolidatedAtCarrierAt` | `set-once` | sí |
| `En tránsito` | `En distribución` | `inTransitAt` | `max(occurredAt)` | sí |
| `Entrega Total` o `Entregada` | `Entregada` | `deliveredAt` | `set-once` | sí |
| `En tránsito` | `Entrega fallida` | ninguno | n/a | **sí** (macro `FAILED`, `stateChange` + `deliveryFailureContext`; sin milestone salvo mapeo terminal) |
| `En tránsito` | `Próximo arribo` | ninguno | n/a | no |
| `En tránsito` | `En tránsito a sucursal de distribución` | ninguno | n/a | no |
| `Anulada` | `Anulada` | ninguno | n/a | **sí** post-gate listable → `CANCELLED`; pre-gate persist only |
| `Suspendida` | cualquier `subStatus` | ninguno en v1 | n/a | no |
| `Cambios en la entrega` | cualquier `subStatus` | ninguno en v1 | n/a | no |
| `En devolución` | cualquier `subStatus` | ninguno en v1 | n/a | no |

### 5.4 Regla explícita para intentos fallidos (última milla)

**Spec:** `plan.md` § B.8 · **Implementación API:** `plan.md` B.8.

**Reintentables (mayoría OOLL):** no abren `MilestoneKey` ni timestamp nuevo.

Regla:
- par `En tránsito` + `Entrega fallida` → API persiste `PackageEvent`, **emite** `PACKAGE_NEW_MILESTONE` con `canonicalStatus: FAILED`, `subStatus: Entrega fallida`.
- incluir `deliveryFailureContext` en payload projector cuando el raw carrier trae datos:
  - `reasonCode` — motivo pre-establecido (obligatorio si hay falla tipificada)
  - `reasonLabel` — opcional, normalizado para UI
  - `comment` — opcional; **persistir solo si llega** (texto libre del operador)
- Andreani `GestionTelefonica 50/51` entra en esta categoría por defecto.
- solo aplica con package en tramo `LAST_MILE` (post-`oeLaunchAt`).
- entrega exitosa posterior sigue abriendo `deliveredAt` + `MilestoneKey: DELIVERED` (macro `DELIVERED` reemplaza macro `FAILED`).

**Terminales (fase 2 B.8.4):** subset de mapeos `Entrega fallida` marcados en `CarrierStatusMapping` → además `milestoneKey: FAILED`.

### 5.5 Condición exacta de emisión a outbox

API emite `PACKAGE_NEW_MILESTONE` al projector cuando:

1. evento raw traducido a par `(status, subStatus)` válido,
2. se cumple **projection gate** (Block 1.11 / `domain-model.md` §5),
3. hay delta efectivo vs watermark,
4. persistencia transaccional completa.

**OOLL v1 — modos de emisión:**

| Modo | Cuándo | `MilestoneKey` | Payload |
|------|--------|----------------|---------|
| Checkpoint | Admisión, consolidación, distribución | — | `operationalTimestamps` + `stateChange` (`IN_TRANSIT`) |
| Entrega fallida reintentable | `Entrega fallida` | — | macro `FAILED` + `stateChange` + `deliveryFailureContext` |
| Entrega fallida terminal | `Entrega fallida` (mapeo) | `FAILED` | idem + `milestoneKey` |
| Entregada | `Entregada` | `DELIVERED` | `deliveredAt` + `stateChange` |
| Cancel / Anulada | `Anulada` post-gate listable | `CANCELLED` | `stateChange` (`CANCELLED`) |

**Regla general:** si OOLL cambia `subStatus` y/o timestamp respecto al último valor proyectado → **emitir** (si post-gate listable). Macro `IN_TRANSIT` en checkpoints; `FAILED` en fallo reintentable; `DELIVERED` / `CANCELLED` en cierres.

Casos v1 sin emisión projector (persist API only): `Anulada` **pre-gate listable**, `Suspendida`, `En devolución` — ver `domain-model.md` §4.3.

### 5.6 Payload hacia projector

Para eventos OOLL v1:
- `stateChange.canonicalStatus` — `IN_TRANSIT` (checkpoints), `FAILED` (entrega fallida), `DELIVERED`, `CANCELLED`.
- `stateChange.subStatus` — granularidad operativa; actualizar en cada aviso relevante.
- `operationalTimestamps` / `milestoneDeltas` — solo keys con delta según matriz 5.3.
- `deliveryFailureContext` — entrega fallida en última milla (Block 5.4).
- `cancellationContext` — opcional v1; cancel manual / enriquecimiento `Anulada`.

Ejemplos:
- `Recepcionado por OOLL` → `IN_TRANSIT` + `pickedUpByCarrierAt`.
- `Entrega fallida` reintentable → `FAILED` + `subStatus` + `deliveryFailureContext` (sin `MilestoneKey`).
- `Entrega fallida` terminal → `FAILED` macro + `milestoneKey: FAILED`.
- `Anulada` post-gate → `CANCELLED` macro + milestone.
- `Entregada` → `DELIVERED` + `deliveredAt` + `milestoneKey: DELIVERED`.

### 5.7 Alcance explícito de v1

- No existe timestamp `failedDeliveryAt`; entrega fallida reintentable usa macro `FAILED` sin `MilestoneKey`.
- Fallo terminal: `milestoneKey: FAILED` solo en mapeos marcados (B.8.4).
- **Sí** existe proyección projector para entrega fallida (`stateChange` + `deliveryFailureContext`).
- `Próximo arribo` y similares: emitir si cambia `subStatus` proyectado (sin timestamp).
- `Anulada`: `CANCELLED` automático post-gate listable; pre-gate persist only.
- `Suspendida`, `Cambios en la entrega`, `En devolución`: persist API; emisión projector diferida.
- Cancel manual declarativo: gate B.8.5 (sin mapeo carrier aún).

### 5.8 Resolución de package y política UNMATCHED (FINAL — 2026-06-23)

**Contexto:** el tópico `andreani-events` notifica cambios de estado para **todos** los `trackId` Andreani con los que Fravega negocia contrato, no solo bultos del MVP warehouse. Muchos avisos llegan **antes** de que exista bulto canónico en API (ej. `AltaAutomatica`, `AltaInterna`, `AltaRemota` — Andreani ya tiene tracking; EnvioPack aún no imprimió etiqueta con `lastMileId`). Otros `trackId` pertenecen a flujos fuera del MVP (retail, HD sin CD, etc.) y nunca se reconciliarán por LPC.

**Decisión v1 (MVP):** política **forward-only** sobre **MATCH**; evidencia **UNMATCHED** persistida para reproceso **v2**. Sin draft ni `Package` sintético desde tracking Andreani.

#### 5.8.1 Flujo de ingestión (por evento)

```
andreani-events (Fenix)
  → validar payload (§5.1)
  → upsert idempotente inbox (externalEventId)
  → resolver package por idAndreani (§5.8.2)
       ├─ MATCH  → procesar dominio + outbox (§5.8.3) → inbox PROCESSED
       └─ NO MATCH → inbox UNMATCHED (§5.8.4) → fin (HTTP 200, no error)
```

- **Prohibido v1:** crear `Package`, `DRAFT_BASE` ni `PackageIdentifierLink` solo desde `idAndreani`.
- **Prohibido v1:** emitir `PACKAGE_NEW_MILESTONE` en UNMATCHED.
- **Prohibido v1:** replay automático de filas UNMATCHED al materializar `lastMileId` en Block 3 (diferido a v2).

#### 5.8.2 Criterio MATCH (v1)

Un evento Andreani tiene **MATCH** cuando se resuelve **un** package canónico que cumple **todas** las condiciones:

| # | Condición |
|---|-----------|
| 1 | Lookup por `payload.idAndreani` en `package_identifier_links` con `identifier_type = LAST_MILE` **o** en `shipments` con `trail_type = LAST_MILE` y `tracking_number = idAndreani` vinculado vía `package_shipments`. |
| 2 | Package con identidad **canónica** (`EP...N-0x` u origen proyectable v1); **nunca** `DRAFT_BASE` (`EP...N` sin secuencia). |
| 3 | Membership activa en tramo `LAST_MILE` (`package_shipments` con `entered_at` set y `exited_at` null). |
| 4 | `package_milestones.oe_launch_at` presente (frontera warehouse cerrada en modelo MVP warehouse-first). |

Si el lookup devuelve 0 o >1 package ambiguo → **NO MATCH** (v1: tratar como UNMATCHED; log métrica `ambiguous_match`).

**Nota timing:** eventos como `AltaAutomatica` que ocurren antes de que Block 3 cree el link `LAST_MILE` serán **UNMATCHED** en v1 aunque más tarde el mismo `trackId` aparezca en un journey LPC. La última milla se construye **solo con eventos posteriores al primer MATCH** (tradeoff aceptado: timeline OOLL menos fina en tramo inicial).

#### 5.8.3 Procesamiento MATCH (v1 — avance de estado)

Cuando hay MATCH, en **una transacción**:

1. Traducir raw → `(status, subStatus)` vía `CarrierStatusMapping` (§5.1).
2. Evaluar matriz §5.3 sobre el par estandarizado.
3. Persistir según política Block 7 / `domain-model.md` §1.1:
   - `PackageMilestones` (`set-once` / `max` según campo),
   - `Package` / `Shipment` (`canonicalStatus`, `subStatus`),
   - `PackageEvent` append-only (`source = ANDREANI`, `eventName` derivado del raw).
4. Si aplica emisión (§5.5): construir `PACKAGE_NEW_MILESTONE` incremental, `return { event }` → interceptor → outbox (watermark filtro **B.2.3**).
5. Marcar inbox `PROCESSED` + `processed_at`.

**Estado código (2026-06-23):** §5.8.1–5.8.4 coded en API. `ConsumeAndreaniEventsUseCase` orquesta; persistencia en `repositories/andreani/` (`IAndreaniRepository`: inbox, match, apply, projection read). Emisión `{ event }` → outbox vía interceptor. **Fixes:** seeder compound keys; contrato `milestoneDeltas` API↔projector. **Pendiente B.2.3:** filtro delta vs `package_projection_watermarks` antes de emitir (repo + delta service **B.2.2** ✅).

**Alcance:** solo el **evento actual**; no consultar ni re-aplicar UNMATCHED históricos del mismo `idAndreani`.

Casos sin emisión projector pero con persist API (matriz §5.3 “no”): inbox `PROCESSED`, sin `{ event }`.

#### 5.8.4 Persistencia UNMATCHED (v1)

Cuando **no** hay MATCH:

| Acción | v1 |
|--------|-----|
| Guardar raw en `inbox_events` | **Sí** (idempotente por `external_event_id`) |
| `status` | **`UNMATCHED`** (extender enum `EventStatus` en schema API) |
| `event_type` | `ANDREANI_EVENTS` |
| `raw_payload` | Payload Andreani completo (`idAndreani`, `evento`, `motivo`, `fechaHora`, …) |
| Mutar `packages` / milestones / outbox | **No** |
| Respuesta HTTP consumer | **200** con `{ id, status: 'UNMATCHED' }` — no es fallo retryable |

**Índice (migration incremental `20260623120000_add_inbox_andreani_unmatched_index`):**
```sql
CREATE INDEX ix_inbox_events_andreani_unmatched
  ON inbox_events (created_at DESC)
  WHERE event_type = 'ANDREANI_EVENTS' AND status = 'UNMATCHED';
```

Consulta v2 (indicativa): filtrar `UNMATCHED` por `raw_payload->>'idAndreani'` o job de backfill por ventana temporal.

#### 5.8.5 v2 — reproceso UNMATCHED (fuera de MVP, diseño reservado)

- Job/worker que, ante nuevo MATCH de un `idAndreani`, re-lea filas `UNMATCHED` del mismo tracking y aplique §5.8.3 en orden `occurred_at`.
- Alternativa: export/archivo antes de TTL de partición si el replay no está listo.
- **No bloquea** cierre MVP B.5: v1 solo requiere persistir UNMATCHED.

#### 5.8.6 Particionado y retención (Block 13)

Post-MVP, `inbox_events` y `outbox_ingestor` se particionarán por tiempo; particiones antiguas se **detach/drop** (ver `blocks-projector.md` Block 13, RFC2 §8).

Implicaciones:

| Tema | Regla |
|------|-------|
| Volumen UNMATCHED | Tolerado en MVP; no genera basura en tablas de dominio (`packages`, `package_events`). |
| Reproceso v2 | Debe ejecutarse dentro de la ventana de retención de particiones `inbox_events` **o** vía export previo al drop. |
| Outbox | Solo eventos MATCH generan filas; particiones outbox siguen política relay existente. |

#### 5.8.7 Métricas operativas mínimas (v1)

- `andreani_events_total` (por resultado: `PROCESSED`, `UNMATCHED`, `FAILED`)
- `andreani_unmatched_rate`
- `andreani_match_latency` (opcional: desde `occurred_at` del primer MATCH)
- `andreani_ambiguous_match_total`

#### 5.8.8 Archivos de implementación (B.5)

**Patrón:** un directorio `repositories/{carrier}/` por operador OOLL (plantilla Andreani para integraciones futuras).

| Pieza | Ubicación | Status |
|-------|-----------|--------|
| Shared OOLL engine (matrix + watermark + outbox) | `application/oollProcessing/oollProcessing.service.ts` — **§5.9** | **DONE** |
| Orquestación ingest Andreani (thin facade) | `consumeAndreaniEvents.useCase.ts` | **DONE** |
| Integración Andreani (inbox + match + apply + projection) | `repositories/andreani/andreani.repository.ts` | **DONE** |
| Mapper Andreani | `repositories/andreani/andreani.repository.mapper.ts` | **DONE** |
| Traducción estado | `statusMappings` + `OollProcessingService` | **DONE** |
| Schema `EventStatus.UNMATCHED` | `schema.prisma` + migrations | **DONE** |
| Simulador local Fenix consume | `simulateAndreaniEvents.js` | **DONE** |
| Tests OOLL engine | `test/unit/ingestor/oollProcessing.service.spec.ts` | **DONE** |
| Tests Andreani facade | `test/unit/ingestor/consumeAndreaniEvents.useCase.spec.ts` | **DONE** — `ambiguous_match` deferred |

### 5.9 `OollProcessingService` extraction (multi-carrier readiness)

**Status:** **DONE** (2026-06-25 — E2E `testing-orchestrator` run `2026-06-25T16-33-08-710Z`, `status: passed`, `softFailureCount: 0`).  
**Plan tracker:** `plan.md` § **B.5.8** (status only; this section is the canonical spec).  
**Goal:** Extract carrier-agnostic OOLL processing (Block 5.3 matrix, watermark delta, listable gate, outbox payload assembly) into a reusable application service. Each logistics operator (Andreani today; OCASA, ClicPaq, MOOVA, etc. later) keeps **only** wire-format adaptation, package MATCH/UNMATCHED, and evidence persistence in its own consumer + repository.

#### 5.9.1 Problem (resolved in B.5.8)

`ConsumeAndreaniEventsUseCase` **previously** owned:

1. Andreani-specific inbox dedupe and MATCH orchestration (acceptable per carrier).
2. Andreani raw-key construction (`evento`, `motivo`, `subMotivo`) (carrier-specific).
3. **Shared** concerns embedded in the same class:
   - `CarrierStatusMapping` resolution → standardized `event` / `subStatus` / `projectionMilestoneKey`
   - Block **5.3** processing matrix (`resolveOollProcessingPlan`)
   - `PackageListableGateService` gating
   - `PackageProjectionDeltaService` watermark filter
   - `PACKAGE_NEW_MILESTONE` payload build (`stateChange`, `milestoneKey`, `operationalTimestamps`, `deliveryFailureContext`)
   - Deterministic `externalEventId` for projector idempotency

`carrier_status_mappings` is already multi-source (seed loads Andreani, OCASA, ClicPaq, FASTTRACK, etc.), but **only Andreani has a Fenix consumer**. Copy-pasting the use case for each operator would have duplicated the matrix and watermark logic — **B.5.8 extracted `OollProcessingService` to prevent that drift**.

#### 5.9.2 Non-goals (this refactor)

- Adding OCASA / MOOVA / ClicPaq consumers (separate work per operator after extraction).
- UNMATCHED replay v2 (Block 5.8.5 — still deferred).
- Changing Block 5.3 business rules, watermark schema, or projector contracts.
- Running E2E or orchestrator validation inside agent sessions — see **§5.9.8**.

#### 5.9.3 Target architecture

```text
[Fenix queue per carrier]     e.g. andreani-events | ocasa-events | moova-events
        │
        ▼
[Carrier notification handler + thin use case]
        │  validate DTO (carrier wire shape)
        │  inbox dedupe (carrier event type)
        │  MATCH / UNMATCHED (carrier repository)
        ▼
[Carrier adapter]             raw payload → { carrierSource, rawStatusKeys[], trackingId, occurredAt, deliveryFailureHints? }
        │
        ▼
[OollProcessingService]       SHARED — mapping lookup, §5.3 plan, apply plan, watermark, listable gate, outbox event
        │
        ├─► [Carrier repository.apply]   milestones + package_events (carrier topic/source)
        └─► returns { inboxStatus, event? }  → EventPublisherInterceptor → outbox (unchanged)
```

**Separation rule:**

| Layer | Owns |
|-------|------|
| **Carrier use case** | Fenix handler registration, DTO/Zod, `externalEventId`, MATCH/UNMATCHED branch, delegate to adapter + service |
| **Carrier repository** | `findMatchByTrackingId`, `applyMatchedEvent`, carrier-specific `package_events` topic/source, inbox `event_type` |
| **Carrier adapter** | Build ordered `rawStatusKeys` candidates from wire payload; map tracking field; optional failure reason fields |
| **`OollProcessingService`** | Everything that depends only on **standardized** status + §5.3 matrix + watermark + projection payload rules |

#### 5.9.4 `OollProcessingService` public contract (indicative)

Location: `shipments-history-api/src/modules/ingestor/application/oollProcessing/`

```typescript
// Names indicative — implement per existing Nest patterns and tokens in ingestor.types.ts

type OollCarrierContext = {
    carrierSource: string;           // CarrierStatusMapping.source, e.g. 'Andreani'
    emissionTrigger: string;         // e.g. 'ANDREANI_OOLL' | 'OCASA_OOLL' (metadata only)
    packageEventSource: string;      // e.g. 'ANDREANI' — for apply input
    packageEventTopic: string;       // e.g. 'andreani-events'
};

type OollResolvedIngress = {
    packageId: string;
    packageExternalId: string;
    occurredAt: Date;
    standardizedStatus: string;
    standardizedSubStatus: string | null;
    projectionMilestoneKey: OollProjectionMilestoneKey | null;
    deliveryFailureContext?: DeliveryFailureContext | null;
    dedupeSeed: string;              // carrier external event id for projection idempotency
};

type OollProcessMatchResult = {
    projectionEvent: EventResponse | null;
    applyInput: ApplyOollMatchedEventInput;  // shared shape or interface implemented by carrier repo
};

// Primary entry after MATCH:
processMatchedEvent(
    ingress: OollResolvedIngress,
    carrierContext: OollCarrierContext,
    projectionPackage: OollProjectionPackageReadModel,
    watermark: PackageProjectionWatermark | null,
): OollProcessMatchResult;

// Shared helpers (may be methods or injected sub-services):
resolveStandardizedStatus(carrierSource: string, rawStatusKeys: string[]): ResolvedCarrierStatus | null;
resolveProcessingPlan(status, subStatus, projectionMilestoneKey): OollProcessingPlan | null;
```

**`OollProjectionPackageReadModel`** must be carrier-neutral (fields already read post-apply in Andreani: milestones, `carrierCode`, `trackingNumber`, `shipmentId`, `trailType`, listable gate inputs).

#### 5.9.5 Per-carrier responsibilities (template)

| Piece | Andreani (v1) | Future operator (e.g. OCASA) |
|-------|-----------------|------------------------------|
| Queue / handler | `andreani-events` | New env + `consumeOcasaEvents.notification` |
| DTO | `AndreaniEventPayloadDto` | `OcasaEventPayloadDto` (wire-specific) |
| Adapter | `evento,motivo,subMotivo` keys; `idAndreani` tracking | e.g. `status,reason` keys; OCASA tracking field |
| Repository MATCH | `findMatchByTrackingId` (§5.8.2 — reuse pattern) | Same LAST_MILE rules; tracking field name differs |
| Repository apply | `andreani-events` topic, `ANDREANI_STATUS_CHANGED` | Operator-specific topic/name |
| Use case thickness | Thin facade after refactor | Copy Andreani facade; **do not** copy matrix |

MATCH criteria (§5.8.2) remain generic: `LAST_MILE` identifier link or active `shipments.tracking_number`, canonical EP external id, `oe_launch_at` set. Only the **tracking id extraction** from payload is carrier-specific.

#### 5.9.6 Behavioral parity (mandatory)

This is a **pure refactor** unless explicitly versioned. Observable behavior must not change for Andreani:

| Contract | Must preserve |
|----------|----------------|
| Fenix consume | Same queue name, payload shape, HTTP status on UNMATCHED |
| Inbox | `ANDREANI_EVENTS`, `PROCESSED` / `UNMATCHED`, dedupe by `externalEventId` |
| Domain apply | Same milestone fields/policies per §5.3 row |
| `package_events` | Topic `andreani-events`, same event name |
| Outbox payload | `stateChange`, optional `milestoneKey`, `operationalTimestamps`, `deliveryFailureContext`, `metadata.emissionTrigger` |
| Watermark | Same skip/emit decisions as pre-refactor |
| Projector | No schema or consumer changes required |

**E2E orchestrator** (`testing-orchestrator` `npm run e2e`) exercises only external boundaries (Fenix POST → API DB → projector SQL/HTTP). It **must pass without orchestrator changes** if parity holds. Validation is a **human / CI** step — not an agent terminal step (§5.9.8).

#### 5.9.7 Unit tests — full rewrite required

Any refactor that extracts `OollProcessingService` **must include a corresponding test rewrite**. Do not leave tests only on the thinned Andreani use case while matrix logic moved untested.

| Test suite | Action |
|------------|--------|
| `test/unit/ingestor/consumeAndreaniEvents.useCase.spec.ts` | **Rewrite/split:** thin use case tests (MATCH, UNMATCHED, dedupe, delegate to service); remove duplicated matrix assertions from monolith |
| `test/unit/ingestor/oollProcessing.service.spec.ts` (new) | **Own** all §5.3 matrix rows, watermark skip/emit, listable gate, `FAILED` macro vs terminal, `DELIVERED` after macro `FAILED`, `Anulada` → `CANCELLED`, delivery failure context |
| Carrier repository specs | Keep MATCH/apply SQL mapping tests on `andreani.repository` (or add when created) |

**Acceptance:** existing Andreani behavioral cases remain covered across the two layers; no reduction in scenario count for matrix + watermark without explicit sign-off.

#### 5.9.8 CRITICAL — agent execution constraint

> **Agents implementing this refactor MUST NOT run terminal commands** (no `npm test`, `npm run build`, `npm run e2e`, migrations, seeds, or shell scripts). Compilation, unit tests, and orchestrator E2E are executed by **humans or CI** only. The agent delivers code + test rewrites + doc updates; verification instructions are written for the operator.

#### 5.9.9 Implementation checklist

| Step | Deliverable |
|------|-------------|
| 1 | Create `OollProcessingService` + types; move `resolveOollProcessingPlan` and projection build from `consumeAndreaniEvents.useCase.ts` |
| 2 | Move mapping lookup helper (or inject `IStatusMappingRepository` + generic `buildRawKeyCandidates` interface) |
| 3 | Slim `ConsumeAndreaniEventsUseCase` to: dedupe → MATCH → adapter → `OollProcessingService` → repository apply → inbox |
| 4 | Register service in `ingestor.provider.ts` / `ingestor.types.ts` |
| 5 | **Rewrite** unit tests per §5.9.7 |
| 6 | Update §5.8.8 table status; mark `plan.md` B.5.8 **DONE** | **DONE** — E2E `2026-06-25T16-33-08-710Z` |

**Suggested files after extraction:**

```text
src/modules/ingestor/application/oollProcessing/
  oollProcessing.service.ts
  oollProcessing.types.ts
  oollProcessing.service.provider.ts
test/unit/ingestor/oollProcessing.service.spec.ts
```

#### 5.9.10 Adding a second carrier (after B.5.8)

1. Seed / mappings already include `source` (verify keys for wire format).
2. Add `repositories/{carrier}/`, notification handler, DTO, adapter implementing raw-key list + tracking extraction.
3. Thin use case clones Andreani flow; inject same `OollProcessingService` with `{ carrierSource, emissionTrigger, packageEventTopic }`.
4. Extend §5.3 or mapping seeds if operator introduces new `subStatus` values not in matrix (returns `null` plan today).
5. Operator-specific simulator + orchestrator step are **optional** follow-ups (not part of B.5.8).

---

## Block 6: Validación Integral de WMS Blocks


**Objetivo:**
Validar que Blocks 2-4 (first mile + LPC + outbox) cubren correctamente los flujos WMS reales y no pierden hitos ni relaciones por bulto.

### 6.1 Cadena validada (warehouse path)

Cadena objetivo confirmada:

1. `ep-ready-to-collect-orders` (draft base `EP...N`)
2. `consumeWmiJourneyAnnouncement` (anuncio de viaje con `journeyUuid`)
3. `WM2_RS_MAIN_OE_LAUNCH` (confirmación física de llegada al warehouse)
4. `WM2_PD_MAIN_CLASSIFICATION` (clasificación por bulto en contenedor)
5. `WM2_PD_MAIN_DISPATCH` con dos subtipos:
  - `DISPATCH_CONTAINER_INTENT`
  - `CONTAINER_PACKAGES_SHIPPED`

### 6.2 Regla de `consumeWmiJourneyAnnouncement` (Block 3 — fase sync)


Cuando llega anuncio de viaje con `wmiExternalId` / `journeyUuid`:

1. Persistir `InboxEvent`.
2. Llamar `WMI_API_WEBHOOK_URL/{wmiExternalId}`.
3. Upsert `Journey` + bulk insert `journey_receipt_items` (staging, **HD + SPU**).
4. Emitir señal Fenix `JOURNEY_RECONCILIATION` `{ journeyId }` al topic `shipments-history-api-to-projector`.
5. Responder **`reconciliationStatus: QUEUED`** — no esperar el loop.

**Milestone en esta fase:** ninguno inline. `journeyAnnouncedAt` lo setea el chunk use case (Block 3.3).

**Prohibido en 6.2:**
- `oeLaunchAt` (pertenece a 6.3 / `WM2_RS_MAIN_OE_LAUNCH`)
- `PACKAGE_NEW_MILESTONE` masivo
- upsert de 20k packages en el HTTP handler
- retener payload WMI completo en memoria fuera de persistencia staging
- excluir SPU del staging

**Resultado:**
- expectativa 1..N packages por journey persistida en staging indexable
- correlación durable por `(sourceSystem="WMI", externalId=wmiExternalId)`
- reconciliación package-level async por loop Fenix (claim + lock, Block 3.2)

Ver detalle completo: **Block 3** (schema 3.0, loop 3.2, HD/SPU 3.3, reglas 3.5).

### 6.3 Regla de `WM2_RS_MAIN_OE_LAUNCH`

`WM2_RS_MAIN_OE_LAUNCH` confirma que la colecta anunciada llegó **físicamente** al warehouse. Es el **primer projection gate B.1** y define la **cohorte** de bultos warehouse del journey.

**Nota:** este evento **no** es WMI journey announcement. Correlaciona con `Journey` previamente creado en Block 3 por `entryOrderExternalId` (= `Journey.externalId` / `wmiExternalId`).

**Payload WMS (entrada):**
```json
{
  "locationExternalId": "htyftyrAEdfGwQVwbhURrJ",
  "entryOrderExternalId": "c90c5d7b-8311-4c80-b2aa-b945dff4a10d"
}
```

Reglas:

1. Persistir `InboxEvent` + correlacionar `Journey` por `entryOrderExternalId`.
2. Resolver packages del viaje vía `JourneyPackage` (preferido) o `journey_receipt_items.packageId` reconciliados.
3. Por cada package canónico del journey:
   - Setear **`oeLaunchAt`** (`set-once`) = `eventTimestamp` OE_LAUNCH.
   - **No** modificar `journeyAnnouncedAt` (ya en API desde chunk Block 3).
   - Append `PackageEvent` técnico (`OE_LAUNCH`).
4. Si un bulto del anuncio WMI aún no tiene canónico en API, dejarlo **pendiente** (out-of-order Block 6.6); emitir cuando Block 3 lo materialice + siguiente OE_LAUNCH o job de catch-up.
5. **Emisión outbox (Block 4):** por package con delta de proyección y gate satisfecho. **Bootstrap warehouse:**
   - `milestoneDeltas`: **`journeyAnnouncedAt`** (desde API) + **`oeLaunchAt`** (nuevo).
   - `metadata.projectionBootstrap: true`, `emissionTrigger: "OE_LAUNCH"`.
   - Incluir links/metadata ya persistidos en Block 3 (`shipmentLink` LAST_MILE HD, `deliveryLink`, SPU fields).
6. Idempotencia re-ingest: warehouse — `appliedPackageIds` vacío si gate ya aplicado (set-once); multi-gate / OOLL — delta vs watermark reclamado en `package_projection_watermarks`.

**Turno implementación:** ✅ `ConsumeOeLaunchUseCase` + handler Fenix + outbox + tramo transitions + payload `WAREHOUSE_RECEIVED`.

### 6.4 Regla de `WM2_PD_MAIN_CLASSIFICATION`

`WM2_PD_MAIN_CLASSIFICATION` emite por bulto e incluye usuario de clasificación.

**Objetivos:**
1. Granularidad operacional: `oeLaunchAt → classifiedAt`.
2. Materializar bultos con reconciliación tardía si faltaron en colecta.

**Reglas v1:**
1. Upsert package por `EP...N-0x` si no existe.
2. Registrar `classifiedAt` (`set-once`).
3. Persistir `classifiedByUserId`, LPN contenedor en metadata/`PackageEvent`.
4. **No emitir** `PACKAGE_NEW_MILESTONE` — datos se empaquetan en gate `LAST_MILE_DISPATCHED` (Block 6.5).
5. Si faltan timestamps previos, marcar backfill pendiente sin inventar fechas.

**Status (code):** ✅ `ConsumeWmsPackageClassificationUseCase` — handler `ASSIGN_ITEM_TO_CONTAINER`; env `FENIX_WM2_PACKAGE_CLASSIFIED_EVENT_NAME`; **no** emite outbox.

### 6.5 Regla de `WM2_PD_MAIN_DISPATCH`

Este tópico tiene dos eventos funcionalmente distintos:

1. `DISPATCH_CONTAINER_INTENT`
2. `CONTAINER_PACKAGES_SHIPPED`

#### A) `DISPATCH_CONTAINER_INTENT`

Semántica:
- asigna contenedores a una orden de salida (`externalOrderId`).
- no confirma salida física.

Regla v1:
- persistir relación operacional orden<->contenedores.
- no setear `dispatchedAt`.
- no emitir `PACKAGE_NEW_MILESTONE` salvo que cree identidad/logística nueva imposible de inferir antes (caso excepcional).

#### B) `CONTAINER_PACKAGES_SHIPPED`

Semántica:
- confirma salida física del warehouse.
- trae contenedores y `orderItems` con packages EP.

Reglas v1:

1. Por cada `orderItems[].id` (`EP...N-0x`), upsert y link a route/container.
2. Setear `dispatchedAt` (`set-once`).
3. Emitir outbox gate **`LAST_MILE_DISPATCHED`** con `milestoneKey`, `operationalTimestamps` (`classifiedAt`, `dispatchedAt`), `warehouseContext` (LPN, operario).
4. Si package no estaba consolidado, marcar reconciliación tardía en metadata.

**Status (code):** ✅ `ConsumeWmsDispatchUseCase` + `WmsDispatchRepository` — `DISPATCH_CONTAINER_INTENT` (bulk metadata) y `CONTAINER_PACKAGES_SHIPPED` (SQL atómico multi-package + gate `LAST_MILE_DISPATCHED`). Handlers: `DISPATCH_CONTAINER_INTENT`, `CONTAINER_PACKAGES_SHIPPED`.

### 6.6 Tolerancia a out-of-order (WMS)

Política de consistencia:

1. No descartar evento válido por ausencia de etapa previa.
2. Aceptar materialización tardía con `reconciliationFlags` en metadata (ej. `missingJourneyAnnouncement`, `missingOeLaunch`).
3. No retroceder milestones set-once.
4. Mantener inmutabilidad de evidencia cruda (`PackageEvent` append-only) y resolver vista canónica en reconciliación.

### 6.7 Criterio de éxito para emitir outbox en transiciones críticas

Se emite `PACKAGE_NEW_MILESTONE` por package cuando se cumple:

1. evento fuente válido + idempotencia superada,
2. **projection gate** del camino satisfecho (Block 1.11),
3. **delta de proyección** vs watermark (no solo delta operacional en API),
4. identidad proyectable (`EP...N-0x` canónico; nunca `DRAFT_BASE`),
5. persistencia transaccional completa (`Package`, `PackageMilestones`, `PackageEvent`, links relevantes).

**Bootstrap B.1:** un evento OE_LAUNCH puede abrir proyección con **dos** milestones en `milestoneDeltas` aunque `journeyAnnouncedAt` se persistió días antes en chunk Block 3.

No se emite cuando:

1. solo hay enriquecimiento API sin gate ni delta de proyección (ej. chunk Block 3),
2. evento duplicado sin delta de proyección,
3. transición ya reclamada en watermark (mismo `MilestoneKey`, timestamps, `canonicalStatus`/`subStatus` proyectados),
4. draft `EP...N` sin identidad canónica.

### 6.8 Cierre de D (deviation)

Con estas reglas quedan definidos para Block 6:

1. chain completa y correlaciones WMS,
2. monotonicidad y política out-of-order,
3. criterio formal de emisión por transición crítica.

---

## Block 7: Política de Milestones y Late Events (Config Runtime API)


**Objetivo:**
Persistir y aplicar reglas de escritura de milestones (`set-once`, `max(occurredAt)`, `override`) y política de late events sin convertir v1 en state machine completa.

### 7.1 Principio de diseño
- `app_config` se usa como **control plane** (puntero/version activa).
- Reglas de dominio viven en tabla/versionado propio de dominio.
- Worker API lee reglas desde cache en memoria (no query por evento).

### 7.2 Modelos a persistir

**A) app_config (existente) — puntero activo**
```text
key: milestone_policy_active_version
value: 1
type: number
```

Opcionales runtime:
```text
milestone_policy_refresh_seconds = 60
milestone_policy_strict_mode = true
```

**B) milestone_policy_versions (nuevo, dominio API)**
```sql
id            UUID PK
version       INT UNIQUE NOT NULL
name          VARCHAR(64) NOT NULL
is_active     BOOLEAN NOT NULL DEFAULT FALSE
policy_json   JSONB NOT NULL
created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
updated_at    TIMESTAMPTZ NOT NULL DEFAULT now()
```

`policy_json` (ejemplo):
```json
{
  "readyToCollectAt": {
    "writePolicy": "set-once",
    "allowBackfillOlder": false,
    "allowOverrideWithCorrection": true
  },
  "inTransitAt": {
    "writePolicy": "max",
    "allowBackfillOlder": false,
    "allowOverrideWithCorrection": true
  },
  "deliveredAt": {
    "writePolicy": "set-once",
    "allowBackfillOlder": false,
    "allowOverrideWithCorrection": true
  }
}
```

### 7.3 Mecánica de lectura/aplicación
1. Al iniciar worker: cargar `milestone_policy_active_version` desde `app_config`.
2. Cargar `policy_json` de `milestone_policy_versions` a memoria.
3. Procesar eventos usando cache local O(1).
4. Refrescar cada N segundos (TTL) o por cambio de versión.
5. Si DB/config falla: usar última policy válida en memoria + defaults embebidos.

### 7.4 Semántica v1 final por milestone (A2)

| Milestone | writePolicy | allowBackfillOlder | allowOverrideWithCorrection |
|---|---|---|---|
| `readyToCollectAt` | `set-once` | `false` | `true` |
| `collectedAt` | `set-once` | `false` | `true` |
| `journeyAnnouncedAt` | `set-once` | `false` | `true` |
| `oeLaunchAt` | `set-once` | `false` | `true` |
| `classifiedAt` | `set-once` | `false` | `true` |
| `dispatchedAt` | `set-once` | `false` | `true` |
| `pickedUpByCarrierAt` | `set-once` | `false` | `true` |
| `consolidatedAtCarrierAt` | `set-once` | `false` | `true` |
| `inTransitAt` | `max(occurredAt)` | `false` | `true` |
| `deliveredAt` | `set-once` | `false` | `true` |

Regla general v1:
- `override` solo aplica si el evento marca `isCorrection=true` y la fuente está autorizada por policy.

### 7.5 Late events (v1)
- Default: no retroceder milestones ya establecidos.
- Excepción de rollback: solo con sobre de corrección y gates de validación.
- Intentos fallidos reintentables (ej. `GestionTelefonica 50/51`): macro `FAILED`, actualizan subStatus/evento; milestone de grilla sin cambio salvo mapeo terminal.

**Sobre de corrección obligatorio (rollback envelope):**
```json
{
  "isCorrection": true,
  "correctionReasonCode": "MISSCAN|DUPLICATE_EVENT|MANUAL_FIX|RETURN_CONFIRMED",
  "correctionRefEventId": "external-event-id-original",
  "correctionSource": "WMS|ANDREANI|ENVIOPACK"
}
```

**Gates v1 para permitir rollback:**
1. `isCorrection=true`
2. `correctionSource` en allowlist
3. `correctionReasonCode` permitido para el milestone
4. Ventana temporal válida (`occurredAtUtc` dentro de `rollbackWindowHours`)

**Matriz v1: retroceso por milestone**

| Milestone | allowRollback | allowedRollbackReasons | Notes |
|---|---|---|---|
| `readyToCollectAt` | `true` | `MISSCAN`,`DUPLICATE_EVENT`,`MANUAL_FIX` | Puede corregirse por reproceso temprano |
| `collectedAt` | `true` | `MISSCAN`,`DUPLICATE_EVENT`,`MANUAL_FIX` | Corrección operativa de first mile |
| `journeyAnnouncedAt` | `true` | `DUPLICATE_EVENT`,`MANUAL_FIX` | WMS/WMI puede rectificar anuncio |
| `oeLaunchAt` | `true` | `MISSCAN`,`MANUAL_FIX` | Evento físico; rollback restringido |
| `classifiedAt` | `true` | `MISSCAN`,`MANUAL_FIX` | Escaneo/operación corregible |
| `dispatchedAt` | `true` | `MISSCAN`,`MANUAL_FIX` | Despacho mal imputado corregible |
| `pickedUpByCarrierAt` | `true` | `DUPLICATE_EVENT`,`MANUAL_FIX` | Corrección por OOLL |
| `consolidatedAtCarrierAt` | `true` | `DUPLICATE_EVENT`,`MANUAL_FIX` | Corrección por OOLL |
| `inTransitAt` | `true` | `DUPLICATE_EVENT`,`MANUAL_FIX` | Campo `max(occurredAt)`; rollback solo por corrección |
| `deliveredAt` | `true` | `MISSCAN`,`RETURN_CONFIRMED`,`MANUAL_FIX` | Terminal crítica; rollback muy restringido |

**Regla adicional para terminal (`deliveredAt`):**
- Si rollback aprobado, emitir evento compensatorio en outbox con razón explícita.

**Configuración runtime recomendada:**
- `rollback_window_hours = 72`
- `rollback_allowed_sources = ["WMS","ANDREANI"]`

### 7.6 Impacto en emisión outbox
- La policy se aplica antes de persistir milestones en API.
- Emisión al projector: policy + **projection gate** (Block 1.11) + **watermark** de proyección.
- Solo se emiten deltas efectivos de proyección en `milestoneDeltas`.
- Idempotencia por `(source, externalEventId)` con hash incluyendo `emissionTrigger` (Block 1.7).

### 7.7 Riesgos y mitigación
- Riesgo: policy inconsistente entre pods -> mitigación: version única activa + refresh TTL corto.
- Riesgo: cambios abruptos de policy -> mitigación: versionado inmutable + rollout por versión.
- Riesgo: costo DB por evento -> mitigación: cache local, sin lectura per-event.

---

## Block 8: Reconciliación Alternativa No-LPC via TMS Lookup API (B.2)


**Objetivo:**
Conciliar packages que no pasan por warehouse/LPC usando endpoint TMS que resuelve relaciones entre `EP...N`, tracking OOLL y shipping id comercial.

**Premisa de orden de implementación:**
- Primero cerrar B.1 (warehouse-first) con alta confianza.
- Luego habilitar B.2 solo para draft packages sin reconciliación LPC.

### 8.1 Endpoint externo (TMS)

Consultas soportadas (mismo endpoint con query param):
- por tracking OOLL o número de reserva (`12812991123`, `36000000002`)
- por EP base (`EP012933331N`)

Respuesta `200`:
- lista de items con `courier_reference` (puede venir secuenciado `EP...N-01`)
- `shipping_id` comercial
- `courier.name` + `courier.tracking_id`

Respuesta `404`:
- no encontrado

### 8.2 Cuándo consultar (gating)

Consultar TMS solo si se cumplen todas:
1. package está en `identity_state = DRAFT_BASE`
2. no hubo reconciliación LPC en ventana de espera
3. package tiene actividad last-mile que requiere correlación
4. intentos de lookup por package debajo de límite

### 8.3 Ventanas temporales v1

- `lpc_wait_window_minutes = 180` (espera LPC antes de lookup)
- `lookup_retry_backoff = [15m, 60m, 6h, 24h]`
- `lookup_max_attempts = 4`
- `lookup_jitter_ms = 250..1500`

### 8.4 Protección anti-DDOS

- Rate limit global por worker (token bucket)
- Concurrency cap por host TMS
- Circuit breaker por errores 5xx/timeouts
- Cache por clave de consulta (TTL)
- Dedupe de requests concurrentes por misma clave

### 8.5 Modelo persistente recomendado

**A) package_lookup_jobs (nuevo)**
```sql
id                UUID PK
package_id        UUID NOT NULL
lookup_key        VARCHAR(128) NOT NULL
lookup_type       VARCHAR(32) NOT NULL   -- EP_BASE | OOLL_TRACKING | RESERVATION
status            VARCHAR(32) NOT NULL   -- PENDING | IN_PROGRESS | DONE | NOT_FOUND | FAILED | EXHAUSTED
attempt_count     INT NOT NULL DEFAULT 0
next_attempt_at   TIMESTAMPTZ
last_error        TEXT
last_response     JSONB
created_at        TIMESTAMPTZ NOT NULL DEFAULT now()
updated_at        TIMESTAMPTZ NOT NULL DEFAULT now()

UNIQUE(package_id, lookup_key, lookup_type)
INDEX(status, next_attempt_at)
```

**B) package_identifier_links (existente)**
- insertar vínculos derivados del lookup:
  - `EP_BASE` -> `EP...N`
  - `EP_FULL` -> `EP...N-01..N-k`
  - `LAST_MILE` -> tracking OOLL
  - `SHIPPING_ID` -> `128-...`

**C) package metadata (existente)**
- actualizar:
  - `identity_state = RESOLVED_BY_TMS_LOOKUP`
  - `resolution_source = TMS_LOOKUP`

### 8.6 Regla de canonización por lookup

- Si TMS devuelve `courier_reference` secuenciado, crear/enlazar `EP...N-01..N-k`.
- Si ya existen packages canónicos por LPC, lookup solo enriquece identificadores (no duplica, no reemplaza identidad).
- Si solo existe draft base, lookup puede promover a canónico con alias explícito.

### 8.7 Emisión a outbox

- Gate **distinto** al bootstrap warehouse B.1 (Block 1.11).
- Emitir `PACKAGE_NEW_MILESTONE` solo si lookup produjo delta de **proyección** (watermark) o identidad canónica utilizable bajo gate Block 8.
- Si lookup solo confirma datos ya persistidos o ya proyectados, no emitir.

### 8.8 Riesgos y mitigación

- Riesgo: exceso de tráfico a TMS -> gating + backoff + rate limit + breaker.
- Riesgo: respuestas parciales -> guardar `last_response`, reintento controlado.
- Riesgo: conflicto LPC vs lookup -> prioridad a LPC para canonización; lookup como enriquecimiento.

---

## Flujo End-to-End (Blocks 2-4 + gate OE_LAUNCH)

```
ep-ready-to-collect-orders
  ↓
Block 2: Create Package (draft EP...N) + Shipment FIRST_MILE + PackageEvent
  ↓
Database API: draft Package, milestones.readyToCollectAt — sin proyección
  ↓
(espera)
  ↓
WMI journey announcement (wmiExternalId)
  ↓
Block 3.1 (sync): InboxEvent + fetch WMI + Journey + journey_receipt_items staging (HD + SPU)
  ↓
Emit Fenix → topic shipments-history-api-to-projector
  eventType: JOURNEY_RECONCILIATION { journeyId }
  ↓
Response WMI: { journeyId, reconciliationStatus: "QUEUED" }
  ↓
┌─ Loop Fenix roundtrip (Block 3.2) — persist API only ──────────┐
│ Fenix → API handler JOURNEY_RECONCILIATION                      │
│   1) claim batch PENDING (SKIP LOCKED) → si vacío: STOP         │
│   2) reconcile chunk (Block 3.3): canónicos, journeyAnnouncedAt,│
│      JourneyPackage; HD: LAST_MILE identifier only; SPU: gaps   │
│   3) emit JOURNEY_RECONCILIATION (siempre si hubo claim)        │
│   4) NO outbox / NO PACKAGE_NEW_MILESTONE                       │
└─────────────────────────────────────────────────────────────────┘
  ↓ (cuando claim vacío)
Journey.status = RECONCILED — loop termina
  ↓
(espera — llegada física al CD)
  ↓
WM2_RS_MAIN_OE_LAUNCH (entryOrderExternalId = Journey.externalId)
  ↓
Block 6.3: oeLaunchAt + Outbox gate 1 por package
  milestoneKey: WAREHOUSE_RECEIVED
  operationalTimestamps: readyToCollectAt, journeyAnnouncedAt, oeLaunchAt
  trailTransitions: close FIRST_MILE, open LAST_MILE
  metadata.projectionBootstrap: true
  ↓
Relay Block 4: OutboxEvent → Fenix (PACKAGE_NEW_MILESTONE)
  ↓
Projector: consume → DeliveryHistoryView + current_milestone
  ↓
(post-bootstrap: DISPATCH gate 2, OOLL incremental, DELIVERED gate 4)
```

**Trigger manual (dev/ops):** `POST /ingestor/journeys/:journeyId/reconcile/chunk` → mismo use case / misma señal Fenix que prod.

---

## Decisiones Importantes

| Decisión | Justificación |
|----------|---------------|
| externalId base sin secuencia en Block 2 | Primer evento no sabe multibulto aún; secuencia viene en Block 3 loop |
| Shipment FIRST_MILE creado en Block 2 | Enviopack es primer operador conocido; es un tramo logístico |
| Journey para LPC, no Shipment por viaje | Viaje operativo de colecta ≠ contrato carrier; ver Block 3.0 |
| WMI sync solo persiste + emite señal | Evita OOM/timeout con 1..20k receiptItems en HTTP handler |
| Topic único sync con dos `eventType` | `JOURNEY_RECONCILIATION` (API loop) + `PACKAGE_NEW_MILESTONE` (projector) en `shipments-history-api-to-projector` |
| Loop Fenix roundtrip, no worker/job | Alineado a stack NestJS + Fenix REST; escala con claim + SKIP LOCKED |
| Claim por status, no offset | Robusto ante fallos parciales y reintentos; no usar `OFFSET` en evento |
| Parada al inicio del handler | Emitir señal interna post-chunk; outbox projector **no** en chunk |
| SPU incluido en reconciliación | Futuro Unigis/sucursales; hoy milestones parciales + discovery gaps |
| `journeyAnnouncedAt` persist en chunk, emit en OE_LAUNCH | Persistencia API vs proyección OLAP (Block 1.4) |
| `oeLaunchAt` solo en OE_LAUNCH (6.3) | Gate de llegada física al CD |
| Primer outbox B.1 = gate `WAREHOUSE_RECEIVED` | Bundle timestamps primera milla + transición tramos (`domain-model.md`) |
| Modo OLAP, no realtime por paso intermedio | Solo projection gates emiten al projector |
| 1 draft → N canónicos | WMI trae secuencia; distinto lastMileId por bulto HD |
| HD: identifier LAST_MILE en chunk; Shipment en OE_LAUNCH | Tramo abre en frontera física CD |
| Classification persist-only | Empaquetada en gate `LAST_MILE_DISPATCHED` |
| Block 3 no crea Shipment por Journey | `FIRST_MILE` Block 2; `LAST_MILE` en OE_LAUNCH |
| Outbox post-return + watermark claim en producer | Use case `{ event }` → interceptor → producer tx (watermark + outbox `PENDING`) → relay |
| Outbox per-package + watermark | Lectura delta en gate; claim atómico con outbox en producer; sin capa post-`SENT` |
| Relay / outbox para milestones | Reintentos `PENDING`/`FAILED` con mismo payload; no re-ingest |
| Non-warehouse path = gate distinto | OOLL sin `oeLaunchAt` → Block 5/8; no bootstrap warehouse |
| OOLL Andreani forward-only + UNMATCHED en inbox | MATCH avanza package; sin MATCH → `UNMATCHED` persistido; sin draft por tracking; replay UNMATCHED = v2 (Block 5.8) |

---

## Known Constraints

- EnvioPack es única fuente FIRST_MILE por ahora; mañana pueden haber más
- LPC es único evento de multibulto por ahora; otros tramos pueden tener lógicas distintas
- DeliveryHistoryView se materializa post **OE_LAUNCH** (gate B.1); modo OLAP, no realtime por hito intermedio
- `readyToCollectAt` se empaqueta en gate `WAREHOUSE_RECEIVED` (no proyección desde Block 2 solo)
- Andreani: muchos `trackId` del contrato global quedarán `UNMATCHED` en MVP; timeline OOLL puede omitir hitos previos al primer MATCH (Block 5.8)
- Replay de `UNMATCHED` sujeto a retención de particiones `inbox_events` (Block 13)

---

## What Block 1 Produces

**Output:** Stream de eventos versionados al topic Fenix bajo **projection gates** (Block 1.11), cada uno con:
- `milestoneDeltas`: delta vs watermark de proyección (puede incluir **N hitos** en bootstrap B.1)
- `stateChange` normalizado (`canonicalStatus` presente)
- Snapshot de identidad/links en bootstrap warehouse (`projectionBootstrap: true`)
- Shipment / delivery linkage si aplica
- `metadata.emissionTrigger` para trazabilidad de cohorte
- Chunking transparent si payload > soft cap
- Deduplicación determinística por `(source, externalEventId)`

**Guarantee:** Projector proyecta idempotentemente (apply multi-key `milestoneDeltas`), sin llamadas síncronas a API.

---

## Execution Priority (API Blocks)

**Done (2026-06-22):** 2, 3, 4, 6.3, 6.4, 6.5 (warehouse path).

1. **Block 1 / B.2** — Watermark: **B.2.2** repo + delta service ✅; **B.2.3** lectura en gates + **B.2.4** claim en `FenixOutboxProducer` (`package_projection_watermarks` schema ✅)
2. ~~**Block 2**~~ — ep-ready + `FIRST_MILE`
3. ~~**Block 3**~~ — WMI + reconciliation loop
4. ~~**Block 4**~~ — Outbox + relay
5. ~~**Block 6.3**~~ — OE_LAUNCH gate `WAREHOUSE_RECEIVED`
6. **Projector** — `current_milestone` + consume gates 1–2 (Fase C)
7. **Block 5** — OOLL + gate `DELIVERED` (B.5)
8. ~~**Block 6.4–6.5**~~ — CLASSIFICATION (persist) + DISPATCH (gate 2)
9. **Block 7** — Política milestones runtime
10. **Block 8** — Reconciliación no-LPC (post B.1)
11. **B.6** — E2E fenix-local API → projector

---

## Known Constraints y Next Steps

### Constraints
- EnvioPack es única fuente FIRST_MILE por ahora; mañana pueden haber más
- LPC es único evento de multibulto por ahora; otros tramos pueden tener lógicas distintas
- DeliveryHistoryView: primer snapshot B.1 en OE_LAUNCH; hitos incrementales después
- Block 3 asume LPC es definitivo (no cambia después); si cambia, requiere lógica de reconciliación delta

### Bloques Posteriores (no en este documento)
- **Projector Block 6:** Consumer de Fenix + InboxEvent processor
- **Projector Block 7+:** Materialización lista, detalle, casos externos

---

## Consistency Snapshot (A-E)

Estado de decisiones de arquitectura (no implementación):

- **A (Contrato v1):** CLOSED
- **B (Identidad base vs secuencia):** CLOSED
- **C (Last-mile matriz/emisión):** CLOSED
- **D (Validación WMS):** CLOSED
- **E (Alineación projector por contrato):** CLOSED

Reglas consolidadas:

1. API emite `PACKAGE_NEW_MILESTONE` solo ante **delta de proyección** + **projection gate** (Block 1.11).
2. B.1 warehouse: primer emisión en OE_LAUNCH con batch `journeyAnnouncedAt` + `oeLaunchAt`.
3. `DRAFT_BASE` es interno de API; projector recibe solo identidades canónicas vía gates.
4. Last-mile se resuelve sobre par estandarizado `(status, subStatus)`.
5. `Entrega fallida` reintentable proyecta macro `FAILED` + `deliveryFailureContext`; sin `MilestoneKey` salvo mapeo terminal (B.8).
6. `Anulada` Andreani → `CANCELLED` post-gate listable (`plan.md` § B.8).
7. Visibilidad temprana distinta a B.1 → nuevos consumers/orígenes en API, no drafts en projector.
8. Topic sync: `JOURNEY_RECONCILIATION` = loop API; `PACKAGE_NEW_MILESTONE` = proyección OLAP.
9. Reconciliación LPC incluye SPU y HD; SPU deja gaps explícitos hasta nuevos orígenes.
10. Non-warehouse: OOLL sin `oeLaunchAt` / sin journey → gate Block 5/8, no bootstrap warehouse.
