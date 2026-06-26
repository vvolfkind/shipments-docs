# Building Blocks: shipments-history-projector — Design Spec

> **Status de implementación:** ver **`plan.md`** Fase C. Este archivo es **spec de diseño**; marcadores Status/DONE/PARTIAL en blocks pueden estar desactualizados.

## Overview

**Definiciones de dominio:** ver **`domain-model.md`** (milestones, tramos, estados, gates).

Projector schema + modules alineados a modelo package-centric. Spec de proyección, catálogos, search y list/detail — ver blocks abajo.

**Rol del projector:** consume eventos con `milestoneKey` + `operationalTimestamps`, materializa read models, mantiene `delivery_history_search_index`, sirve listado con `current_milestone` (`MilestoneKey`) y detalle con `segments[]` por tramo.

---

## Blocks (design spec)

Índice — detalle por block abajo. **Qué está hecho:** `plan.md` Ground Truth + Fase C.

| Block | Tema |
|-------|------|
| 1–5 | Schema, DTOs, mocks, módulos |
| 6 | Ingestor `PACKAGE_NEW_MILESTONE` |
| 7–8 | List/detail derivation |
| 9–10 | External scenarios, retire mocks |
| 11 | Search index |
| 12 | Filter catalogs + milestones |
| 13 | Partitioning (deferred) |

---

### Block 1: Alinear Vocabulario

Entidades separadas:
- `Package`: bulto trazable canónico
- `Shipment`: tramo logístico
- `DeliveryView`: agregado visible para backoffice

**Validación:** Schema correctamente modelado. No hay ambigüedad semántica.

---

### Block 2: Definir Contratos de Salida

DTOs:
- `ListShipmentsOutputDto`: grilla backoffice (id, order, shipment, parcel, trackingId, courier, deliveryType, status, subStatus, etd)
- `GetShipmentDetailOutputDto` **v1**: detalle operativo (packages, segments, milestones, items, timeline, summary)
- `GetShipmentDetailOutputDto` **v2** (spec): contrato de **datos normalizados** para que el frontend arme `ShipmentDetailViewModel` vía adapter — ver Block 8 §8.1

**Validación:** DTOs v1 validados con Zod. Lista alineada a grilla mock. Detalle v1 cubre núcleo operativo; gap vs backoffice → Block 8 §8.1 (spec cerrada 2026-06-25), implementación **Block R2**.

---

### Block 3: Exponer Mocks en Controllers

Endpoints activos:
- `GET /delivery-history/shipments` (mock)
- `GET /delivery-history/shipments/:id` (mock)

Controllers:
- `DeliveryHistoryController`: validación de paginación (limit 1-100, offset ≥0)
- Decorator `@mockDeliveryHistoryResponse`: alterna entre mock/real vía `MOCKS_ENABLED`

**Validación:** Frontend puede integrar contra mocks ya. Decorator en place para switch real.

**Pendiente:** Eliminar mocks una vez data real disponible (Block 10).

---

### Block 4: Evolucionar Schema a Package-Centric

Entidades persistidas:
- `Package`: bulto base (externalId, sourceSystem, currentCarrierCode, status, metadata)
- `PackageEvent`: append-only timeline (source, topic, eventName, occurredAt, payload)
- `PackageIdentifierLink`: reconciliación N:1 (identifier → package)
- `PackageMilestones`: snapshot de **timestamps operativos** (11 campos `*At`; ver `domain-model.md` §1)
- `PackageShipment`: relación N:M (package ↔ tramo) con `enteredAt` / `exitedAt`
- `Shipment`: tramo logístico (`trailType`: `FIRST_MILE` | `LAST_MILE` en v1)
- `Journey` + `JourneyPackage`: operaciones internas WMS (viajes, routes)
- `DeliveryHistoryView`: materialización UI (primaryPackage, primaryShipment, packages[], shipments[])
- `LogisticsOperator` + `OperationType`: catálogos de filtros (tablas `logistics_operators`, `operation_types`)

Índices:
- `ix_packages_status`: filtro por estado
- `ix_package_events_timeline`: (packageId, occurredAt DESC) para timeline rápida
- `ix_delivery_history_views_listable_updated_at`: grilla listable
- Partial indexes en inbox: solo PENDING/FAILED accionables

**Validación:** Schema soporta warehouse + flujos externos sin refactor estructural.

---

### Block 5: Ajustar Módulos Proyector

Arquitectura:
- `/repositories/deliveryHistoryView`: lista + detalle
- `/useCases/listShipments`: query repo, mapea a OutputDto (métodos privados en el use case)
- `/useCases/getShipmentDetail`: query repo, read model vía `repository.mapper.toDetail`, OutputDto en métodos privados del use case
- `/controllers/deliveryHistory.controller`: HTTP entry points
- DTOs + entities separadas de Prisma models

**Validación:** Patrón framework (interface/model/mapper/repository/provider) mantenido.

---

### Block 6: Introducir Eventos Reconciliados desde API

**Decisión de integración (RFC2/DDIA):**
- Camino elegido: API emite evento con payload de cambios y altas de package. Projector consume y proyecta localmente.
- Camino descartado como flujo principal: evento señal + pull a API.

**Justificación técnica:**
- RFC2 define separación write-heavy/read-heavy con transporte por eventos y proyección asíncrona, no query síncrona entre servicios.
- DDIA favorece event log como fuente de verdad para derivar read models idempotentes y reproducibles.
- Menor acoplamiento temporal: projector no depende de disponibilidad/latencia de API para cada evento.
- Mejor resiliencia: replay, backfill y recuperación por reconsumo de eventos.
- Menor riesgo operativo: evita thundering herd y polling cascada cuando sube volumen.

**Regla operativa:**
- Flujo normal: event-carried state transfer (payload completo de delta relevante).
- Excepción controlada: signal+pull solo para bootstrap/recovery/backfill puntual.

**Qué existe:**
- `InboxEvent` model: almacena eventos reconciliados (source, aggregateType, aggregateKey, eventType, payload)
- UNIQUE constraint: (source, externalEventId) → deduplicación
- Índices: `ix_inbox_events_poll` (status PENDING/FAILED), `ix_inbox_events_aggregate`

**Topico Fenix:** `shipments-history-api-to-projector`

**Contrato v1 de Evento — Especificación:**

```json
{
  "eventVersion": 1,
  "eventType": "PACKAGE_STATE_CHANGED",
  "externalEventId": "api-pkg-evt-{uuid}",
  "occurredAt": "2026-06-17T14:30:00.000Z",
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
    "subStatus": null,
    "canonicalStatus": "IN_TRANSIT",
    "lastOccurredAt": "2026-06-17T14:25:00.000Z"
  },
  
  "milestoneDeltas": {
    "readyToCollectAt": "2026-06-10T13:33:00.000Z",
    "oeLaunchAt": "2026-06-08T19:37:00.000Z",
    "classifiedAt": "2026-06-11T07:54:00.000Z",
    "dispatchedAt": "2026-06-11T08:18:00.000Z",
    "inTransitAt": "2026-06-17T14:25:00.000Z"
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
    "batchId": "batch-20260617-001",
    "chunkIndex": 0,
    "chunkTotal": 1,
    "reconciliationId": "recon-12345"
  }
}
```

**Campos obligatorios:**
- eventVersion, eventType, externalEventId, occurredAt, source
- package.externalId, package.sourceSystem
- stateChange.status, stateChange.canonicalStatus, stateChange.subStatus
  - *Estandarización ocurre en API via diccionario de traducción de estados; projector recibe normalizado*

**Campos condicionales:**
- milestoneDeltas: al menos uno si hay cambio de hito
- shipmentLink: si existe relación a tramo logístico
- deliveryLink: si existe vínculo a delivery comercial

**Campos opcionales:**
- metadata.batchId: para batch tracking
- metadata.chunkIndex/chunkTotal: si payload se particionó por tamaño

**Responsabilidades:**
- **API:** diccionario carrierCode:rawStatus → canonicalStatus. Traducción ocurre durante reconciliación. Evento viaja normalizado.
- **Projector:** consume evento ya normalizado. No retraslada ni recalcula canonicalStatus.

**Semántica de idempotencia:**
- Clave única: (source, externalEventId)
- Projector rechaza duplicado sin error (idempotente)

**Semántica de chunking (si payload > 512 KB soft cap):**
- API particiona eventos
- Todos comparten externalEventId base (reconciliationId determinístico)
- Projector bufferiza chunks por reconciliationId
- Proyecta solo cuando chunkIndex + 1 == chunkTotal
- TTL 5 min en buffer; chunks incompletos → DLQ después

**Qué existe (código — 2026-06-25):**
- `ConsumePackageNewMilestoneNotificationHandler` + `ApplyPackageNewMilestoneUseCase` (dedupe + tx)
- Proyección incremental: OE_LAUNCH, dispatch, OOLL (timestamps, `stateChange`, `deliveryFailureContext`)
- `current_milestone` desde `milestoneKey` (**C.3 + C.7** — 5 keys, terminales)
- `DeliveryHistoryView` bootstrap con `isListable: true` hardcodeado
- `listShipments` expone `currentMilestone`
- **Block 11 / C.5:** `delivery_history_search_index` poblado en tx de proyección (`replaceSearchIndex`)
- **Block 11–12 / C.6:** list `q` + filtros (`courier`, `deliveryType`, `milestone`) vía `DeliveryHistorySearchRepository`

**Qué falta (prioridad):**
- C.2 persistencia: tabla `milestones` (HTTP catalog ✅)
- C.4: `PackageEvent.shipmentId` por tramo activo
- Chunk buffer (si API particiona payload)
- Search **performance** P1 (`search-performance-spec.md`) — lectura optimizada, no contrato HTTP

**Próximo paso:** C.4 tramo · C.2 persistencia catálogo · SP.1 performance.

---

### Block 7: Construir Proyección de Lista

**Decisión de listabilidad (actualizada 2026-06-23):**
- Un package **solo es listable** después del **primer gate listable** del flujo (MVP warehouse: `WAREHOUSE_RECEIVED` / `OE_LAUNCH`).
- `CANCELLED` / `Anulada` **antes** del gate listable → **no** aparece en grilla (API no emite, o emite con `isListable: false` — MVP: no emitir).
- Futuros flujos sin warehouse pueden usar otro entry gate; la regla genérica se mantiene.
- `isListable` lo decide **API en el contrato de emisión**; projector **persiste** el flag (C.7 rompe hardcode `isListable: true` actual en bootstrap mapper).
- Eliminar lógica asincrónica de marcado; el flag viene con el evento o con bootstrap gate listable.

**Decisión de estado en listado:**
- El **estado visible** en grilla = **`current_milestone`** (`MilestoneKey`: 5 valores — `domain-model.md` §4.1).
- `subStatus` permanece en detalle/timeline para diagnóstico; **nunca** como dimensión de filtro.
- Macro-estado `canonicalStatus`: `IN_TRANSIT` | `FAILED` | `DELIVERED` | `CANCELLED`.
- Vista por **tramo** (`segments[]`): `FIRST_MILE` (pre-`oeLaunchAt`) vs `LAST_MILE` (post-`oeLaunchAt`, incluye CD + carrier).

**Qué existe:**
- `listShipments` use case → `DeliveryHistorySearchRepository.searchListable()` (con/sin `q`)
- Query filtra `WHERE isListable = true` (+ filtros milestone/courier/deliveryType)
- Order by `updatedAt DESC`
- Bootstrap projection setea `displayId`, `courier`, `deliveryType`, `trackingNumber`, `orderId` en OE_LAUNCH
- `current_milestone` derivado y persistido (**C.7** ✅)

**Qué falta:**
- Ruleset para derivar `parcel` (primary package `externalId`)
- Ruleset para derivar `courier` estable (primaryShipment `carrierCode` o `package.currentCarrierCode`)
- Ruleset para derivar `deliveryType` (`HD` / `SPU` desde metadata; default `HD`)
- Atribuir `PackageEvent.shipmentId` al tramo activo en cada proyección (**C.4**)
- Conectar HTTP real (`MOCKS_ENABLED=false`) una vez Block 10 cierre E2E

**Próximo paso:** Derivaciones list + C.4 tramo; Block 10 E2E integrado.

---

### Block 8: Construir Proyección de Detalle

**Análisis de gap:** Block 8 §8.1 — contrato v2 de **datos normalizados** para adapter frontend (`ShipmentDetailViewModel` no es isomórfico al HTTP).

**Decisión de modelo de detalle:**
- La unidad de trazabilidad es el bulto: cada package se expone con su propio estado, timeline y milestones.
- No existe agregación de **estado** entre packages: no hay `canonicalStatus` sintético del grupo.
- La reconciliación comercial (orden, delivery, direcciones, ítems enriquecidos) es contexto opcional en la vista, no el eje de presentación.
- Si frontend necesita agrupar packages relacionados en una vista superior, es responsabilidad de presentación; projector expone packages como entidades propias.

**Qué existe (v1):**
- `getShipmentDetail` use case
- Query fetches: packages[] (con items + events + milestones) + shipments[]
- Output mapea a: summary (totalPackages, totalItems, totalWeightGrams), segments[], packages[], timeline per package
- Archivos: `getShipmentDetail.dto.ts`, `getShipmentDetail.useCase.ts` (mapeo HTTP en métodos privados), `deliveryHistoryView.repository.ts` + `deliveryHistoryView.repository.mapper.ts` (`toDetail`, derivados persistidos p.ej. `volumeCubicCm`)

**Qué falta (composición + contrato v2):**
- Implementar `GetShipmentDetailOutputDto` v2 (Block 8 §8.1)
- Validación: permitir detalle aunque shipments vacío (casos externos only-package)
- Fallback: si primaryShipment null, usar first shipment en relación
- Aggregation logic: si múltiples shipments → ordenar segments[] cronológicamente por tramo
- ETD derivation: si primaryShipment null, usar package milestone (estimated via carrier)
- Lookup alternativo por `displayId` o tracking vía search index (Block 11) además de UUID canónico
- Navegación lista→detalle: `ListShipmentsResultItem.id` **debe** ser el UUID de `DeliveryHistoryView` (no `displayId` / `shipmentId` visible)

**Próximo paso:** Block R2 (evolución DTO detalle); luego casos externos (Block 9).

---

#### 8.1 Contrato de salida — `GetShipmentDetailOutputDto` v2 (SPEC)

**Última revisión:** 2026-06-25 — contrato de **datos**, no de presentación UI.

**Objetivo:** Modificar el DTO de salida de `GET /delivery-history/shipments/:id` para que el frontend (`backoffice-shipments-zone`) pueda reconstruir `ShipmentDetailViewModel` (`@fravega-it/logistic-ds`) mediante un **adapter local**. El projector **materializa y expone datos normalizados**; no reinterpreta semántica de dominio (`domain-model.md`) ni emite copy, colores, labels ni URLs de seguimiento.

**Endpoint:** `GET /delivery-history/shipments/:id`  
**Input:** `id` = UUID de `delivery_history_views.id` (canónico). Resolución por `displayId`/tracking: endpoint o query param separado (Block 11), fuera de este contrato de body.

##### Separación datos vs presentación

| Capa | Dueño | Ejemplos |
|------|-------|----------|
| **API (projector)** | Datos semánticos, ISO datetimes, códigos (`courier`, `deliveryType`), totales numéricos | `logistics.destination.recipientName`, `summary.totalWeightGrams` |
| **Frontend (zone)** | Copy, i18n, colores, emojis, `key`/`label` de UI, tonos, fechas formateadas, links externos | `logistics[].title: '🏢 Origen'`, `quickActions`, `"3.2 kg"` |

El mock inline en `backoffice-shipments-zone/.../[id]/page.tsx` es **target de presentación**, no shape del HTTP. Debe reemplazarse por `GET /delivery-history/shipments/:id` + `mapDetailToViewModel()`.

**Fuera de scope API (explícito):**

- `quickActions`, `trackingUrl`, URLs por carrier — el frontend infiere desde `courier` + `trackingId` con config local (sin catálogo backend salvo decisión futura desacoplada).
- `title`, `logisticsTitle`, `shipmentInfoTitle` — copy fijo del zone.
- `logistics[].key`, `title`, `color`, `fields[].label`, `emphasized`, `italic` — config declarativa del adapter.
- `shipmentInfo.leftFields` / `rightFields` / `summary` con `label`/`tone` — layout del adapter.
- Timeline `title`, `highlighted`, `date` formateada — derivados en adapter desde `eventName`, `status`, `occurredAt`.

##### Reglas de serialización (obligatorias)

| Regla | Descripción |
|-------|-------------|
| **Shape estable** | Toda key definida en el schema v2 **siempre** aparece en el JSON de respuesta. |
| **Valor ausente** | Si el dato no existe o el enriquecimiento comercial no está disponible → **`null`**, nunca omitir la key del contrato. |
| **Sin invención** | Projector no fabrica datos comerciales (cliente, direcciones, seller, peso por ítem comercial) para “rellenar” la UI. |
| **Sin presentación** | No enviar `label`, `color`, `emphasized`, `tone`, URLs ni strings con unidades (`"3.2 kg"`). Valores escalares en unidades canónicas (gramos, cm³) o códigos. |
| **Derivados operativos** | Campos calculables solo desde datos ya proyectados (totales, `statusItems` opcional) se computan en use case; si el insumo operativo falta, el escalar derivado es `null` y las colecciones derivables son `[]`. |
| **Enriquecimiento comercial** | Campos marcados *comercial*: `null` hasta fuente persistida/proyectada (evento API, `deliveryLink`, TMS, metadata WMS). No bloquea 200 del detalle operativo. |
| **Por bulto vs vista** | Estado, timeline y milestones son **por package**. Ítems comerciales en `packages[].items[]`. Origen/destino/devolución a nivel `DeliveryHistoryView`. |

##### Campos v1 — se mantienen (mismos nombres; valores `null` si no hay dato)

| Campo | Tipo | Fuente | Notas |
|-------|------|--------|-------|
| `id` | `string` (uuid) | `DeliveryHistoryView.id` | Acceso canónico; lista debe navegar con este valor |
| `shipmentId` | `string` | `displayId` | Identificador visible; **no** usar como `:id` del GET |
| `orderId` | `string \| null` | view | Comercial si reconciliado |
| `externalOrderId` | `string \| null` | view | Comercial si reconciliado |
| `trackingId` | `string \| null` | primary shipment / package | Contrato UI; no id de acceso |
| `courier` | `string \| null` | view / primary shipment | |
| `deliveryType` | `string \| null` | view | `HD` / `SPU` / etc. |
| `status` | `string` | package primario o view | `canonicalStatus` visible (no milestone de grilla) |
| `subStatus` | `string \| null` | idem | Diagnóstico; no filtro |
| `etd` | `string` (datetime) \| `null` | primary shipment | |
| `lastOccurredAt` | `string` (datetime) \| `null` | view | |
| `summary` | `object` | derivado | Siempre presente; ver abajo |
| `segments` | `array` | shipments[] | Siempre presente; `[]` si vacío |
| `packages` | `array` | packages[] | Siempre presente; mínimo 1 en vistas listables |

`summary` (siempre presente):

| Campo | Tipo | Derivación |
|-------|------|------------|
| `totalPackages` | `number` | `packages.length` |
| `totalItems` | `number` | sum `items[].quantity` |
| `totalWeightGrams` | `number \| null` | sum `packages[].weightGrams` si todos conocidos; si alguno `null` → `null` |
| `totalVolumeCubicCm` | `number \| null` | sum volúmenes si todos conocidos; si no → `null` (**nuevo v2**) |

`packages[]` (v1 + extensiones v2):

| Campo | Tipo | Fuente | Notas |
|-------|------|--------|-------|
| `id`, `externalId`, `trackingId`, `courier`, `status`, `subStatus` | v1 | operativo | |
| `weightGrams` | `number \| null` | operativo | |
| `volumeCubicCm` | `number \| null` | dimensiones proyectadas | **nuevo v2**; `null` sin dimensiones |
| `milestones` | `object` | v1 | 11 timestamps `*At` |
| `items` | `array` | v1 + extensión | |
| `timeline` | `array` | v1 | |

`packages[].items[]`:

| Campo | Tipo | Fuente | Notas |
|-------|------|--------|-------|
| `id`, `sku`, `name`, `quantity` | v1 | operativo | |
| `weightGrams` | `number \| null` | comercial / catálogo | **nuevo v2**; `null` sin enriquecimiento |
| `volumeCubicCm` | `number \| null` | comercial / dimensiones | **nuevo v2**; `null` sin enriquecimiento |

##### Campos v2 — datos enriquecidos (nuevos)

| Campo | Tipo | ¿Comercial? | Derivación / fuente | Si no hay dato |
|-------|------|-------------|---------------------|----------------|
| `statusItems` | `array` | No | Opcional: `{ packageExternalId, status, subStatus }` por package (espejo operativo; frontend puede derivar de `packages[]`) | `[]` |
| `logistics` | `object` | Sí | Ver estructura semántica abajo | Objeto presente; hijos `null` |
| `shipmentInfo` | `object` | Mixto | Campos planos operativos/comerciales — ver abajo | Objeto presente; hijos `null` |

**No incluir en v2:** `quickActions`, `trackingUrl`, bloques UI con `label`/`color`/`emphasized`.

`statusItems[]` (opcional — frontend puede derivar de `packages[]`):

```typescript
type StatusItemDto = {
  packageExternalId: string;
  status: string;
  subStatus: string | null;
};
```

##### Tipos compartidos

```typescript
type AddressDto = {
  street: string | null;
  streetNumber: string | null;
  floor: string | null;
  apartment: string | null;
  postalCode: string | null;
  city: string | null;
  province: string | null;
  country: string | null;
  formatted: string | null; // solo si la fuente ya lo trae normalizado; no concatenar en projector
};

type LogisticsOriginDto = {
  distributionCenterCode: string | null;
  distributionCenterName: string | null;
  address: AddressDto | null;
};

type LogisticsDestinationDto = {
  recipientName: string | null;
  address: AddressDto | null;
};

type LogisticsReturnDto = {
  sellerName: string | null;
  address: AddressDto | null;
  reference: string | null;
};
```

`logistics` (siempre presente — las tres keys obligatorias):

```typescript
{
  origin: LogisticsOriginDto | null;
  destination: LogisticsDestinationDto | null;
  return: LogisticsReturnDto | null;
}
```

Si no hay nodo comercial / fuente para un bloque → el bloque entero es `null`. Si hay bloque parcial → el objeto existe y cada subcampo sin dato es `null`.

El adapter del frontend mapea cada bloque a `ShipmentDetailViewModel.logistics[]` (keys `origen`/`destino`/`devolucion`, títulos, colores, labels) desde **config local** del zone.

`shipmentInfo` (siempre presente — campos planos; **no** duplicar raíz salvo alias documentado):

| Campo | Tipo | Fuente | `null` cuando |
|-------|------|--------|---------------|
| `journeyId` | `string \| null` | Journey / WMS proyectado | Sin journey en view |
| `sortable` | `boolean \| null` | metadata WMS / sorter | Sin señal |
| `routeId` | `string \| null` | dispatch / journey | Sin ruta |
| `committedDeliveryAt` | `string` (datetime ISO) \| `null` | comercial / carrier | Sin compromiso distinto de `etd` raíz |
| `rescheduledDeliveryAt` | `string` (datetime ISO) \| `null` | comercial / OOLL | Sin replanificación |

**En raíz (no repetir en `shipmentInfo`):** `orderId`, `externalOrderId`, `deliveryType`, `courier`, `trackingId`, `etd`, `lastOccurredAt`. El adapter elige columnas izquierda/derecha del mock UI.

Labels de `deliveryType` (ej. `"Home Delivery"`) → catálogo `GET /delivery-history/catalog/operation-types` en el frontend.

##### Capas de implementación (orden sugerido)

| Capa | Campos | Depende de |
|------|--------|------------|
| **A — Operativo** | v1 + `summary.*` + `statusItems` (opcional) + `segments`/`packages`/`timeline` + `volumeCubicCm` desde `Package.dimensions` | Solo read model actual |
| **B — Comercial / logístico** | `logistics.*`, `shipmentInfo.journeyId/routeId/sortable`, `items[].weightGrams/volumeCubicCm` | Ingesta/proyección futura (API `deliveryLink`, TMS, metadata WMS) |

Capa A puede implementarse en projector sin nueva ingesta. Capa B devuelve `null` hasta tener fuente. **No hay capa de presentación en API.**

##### Adapter frontend (referencia)

Responsabilidad del zone (`mapDetailToViewModel`):

- `logistics` semántico → bloques DS con config `detailView.config.ts` (keys, colors, field labels).
- `courier` + `trackingId` → `quickActions` / link seguimiento web (config local por carrier).
- `summary.totalWeightGrams` / `totalVolumeCubicCm` → strings con unidad + `tone`.
- `packages[].timeline` → items DS con `title`/`highlighted`/`date` formateados.
- `deliveryType` código → label vía catálogo operation types.

Lista: `onShipmentClick` debe usar `row.id` (UUID), no `row.shipment` / display id (Block R1 + Block 8 §8.1).

##### Contradicciones revisadas (spec vs docs previos)

| Tema | Doc anterior | Esta spec (2026-06-25) | Resolución |
|------|--------------|------------------------|------------|
| Block 2 “DONE” detalle | Contrato cerrado v1 | v2 extiende contrato | v1 DONE; v2 = Block R2 |
| `LogisticsBlockDto` con `label`/`emphasized` | Presentación en API | Objetos semánticos + `AddressDto` | Eliminado — adapter frontend |
| `quickActions` en DTO | Block 12 / catálogo URLs | Fuera del contrato | Frontend infiere; sin acoplamiento backend |
| `trackingLabel` en `shipmentInfo` | Campo API | Usar `trackingId` raíz | Sin duplicación |
| `shipmentInfo.lastUpdatedAt` | Alias en API | Usar `lastOccurredAt` raíz | Sin duplicación |
| Sin agregación cross-package | No status sintético | `statusItems` + `summary` | OK: espejo por bulto / métricas numéricas |
| Listado `milestone` vs detalle `status` | Grilla `MilestoneKey` | Detalle `status`/`subStatus` por package | OK: filtro vs diagnóstico |
| Enriquecimiento comercial opcional | `null` si no hay fuente | Igual | OK |
| Detalle por `displayId` | Block 11 | UUID canónico + lookup separado | Sin cambio |

**Sin contradicción con `domain-model.md`:** capa HTTP de lectura sobre read model proyectado; no introduce gates ni tramos nuevos.

---

### Block 9: Extender a Casos Externos No Warehouse

**Caso:** Bulto colectado por EP → Andreani ultima milla directo (sin paso por CD).

**Qué falta:**
- Validación en test: timeline útil sin warehouse hitos
- Flag en Package/DeliveryView: `hasWarehouseProcess` (boolean)
- UI fallback: detalle sin CLASSIFICATION, DISPATCH eventos, solo status del carrier
- Search index: indexar tracking Andreani y EP sin hitos WMS (mismas reglas Block 11)
- Filtro `courier=ANDREANI` debe funcionar sin paso por CD

**Próximo paso:** Validar schema + test cases para bultos EP→carrier solo; reindexar en proyección externa.

---

### Block 10: Cerrar Sincronización Real & Apagar Mocks

**Qué falta:**
- API → Fenix queue consumer activo en projector (replay local desde `outbox_ingestor` aceptable en dev)
- Validar pipeline: OE_LAUNCH bootstrap → `DeliveryHistoryView` + search index + list con `q` y sin `q`
- E2E: búsqueda por tracking EP, por orden (cuando reconciliada), listado filtrado por milestone/courier/deliveryType
- Toggles: `MOCKS_ENABLED=false` con rollback a mocks si fallara consumer
- Eventos incrementales post-bootstrap (classification, dispatch, last mile) según API

**Próximo paso:** Replay outbox API → projector; validar list/search/filters sin mocks.

---

### Block 11: Motor de Search PostgreSQL (Search Index)

**Decisión:** El projector deja de ser pasivo en lectura: además de proyectar, **mantiene un índice de búsqueda** optimizado para PostgreSQL (patrón RFC2 §7.3, adaptado a `delivery_history_search_index`).

**Tabla (ya en schema inicial — no nueva migration hasta cerrar modelo):**
```sql
delivery_history_search_index (
    field_value   VARCHAR(256) NOT NULL,  -- normalizado
    view_id       UUID NOT NULL,
    field_type    VARCHAR(32) NOT NULL,
    updated_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (field_value, view_id)
);
-- ix_delivery_history_search_index_prefix: varchar_pattern_ops ✓
-- ix_delivery_history_search_index_view_id ✓
```

**Tipos de campo (`field_type`) — MVP + reservados:**

| field_type | Origen | Cuándo |
|------------|--------|--------|
| `ORDER_ID` | `orderId`, `externalOrderId` | Cuando reconciliación comercial existe |
| `PACKAGE_EXTERNAL_ID` | `package.externalId` (EP…-NN) | Siempre en bootstrap |
| `TRACKING` | `trackingNumber`, `currentTrackingId`, `package_identifier_links` | Cuando hay carrier/EP tracking |
| `CUSTOMER_DNI` | metadata comercial / delivery link | **Reservado** — indexar cuando API reconcilie orden con DNI |

**Normalización (obligatoria antes de insert y antes de query):**
- `trim` + `uppercase`
- Quitar espacios internos en tracking numérico
- Guiones: conservar en EP (`EP012933329N-01`); opcional variante sin guión como fila adicional si mejora match
- Orden Frávega: valor literal normalizado (`128-18733100-12991123`)

**Escritura (misma transacción que proyección — Block 6):**
1. Tras upsert de `DeliveryHistoryView`, **delete** filas previas `WHERE view_id = $id`
2. **insert** batch de tokens normalizados (idempotente por PK)
3. Recalcular `current_milestone` en view (ver regla abajo)
4. Nunca escribir search index fuera del mapper/repository de proyección

**Regla `current_milestone` (columna en `delivery_history_views`):**
- Definición cerrada en `domain-model.md` §4.1 y §4.3.
- **Progreso** (forward-only): `WAREHOUSE_RECEIVED` → `LAST_MILE_DISPATCHED`.
- **Terminales:** `DELIVERED` | `FAILED` | `CANCELLED` — reglas especiales (no pipeline lineal):
  - `FAILED` → `DELIVERED` permitido.
  - `DELIVERED` → `FAILED` prohibido.
  - `CANCELLED` absorbente (nada posterior).
- OOLL reintentable: macro `FAILED` en detalle; **no** avanza `current_milestone` salvo mapeo terminal.
- Implementación: **C.7** — reemplaza `resolveCurrentMilestone` lineal actual (3 keys).

**Índices adicionales (incluir en migration inicial, no migration aparte):**
```sql
-- partial, solo listables
CREATE INDEX ix_dhv_listable_courier
  ON delivery_history_views (courier, updated_at DESC)
  WHERE is_listable = true AND courier IS NOT NULL;

CREATE INDEX ix_dhv_listable_delivery_type
  ON delivery_history_views (delivery_type, updated_at DESC)
  WHERE is_listable = true AND delivery_type IS NOT NULL;

CREATE INDEX ix_dhv_listable_milestone
  ON delivery_history_views (current_milestone, updated_at DESC)
  WHERE is_listable = true;
```

**Decisión HTTP (FINAL — 2026-06-25):** búsqueda y filtros en el **mismo endpoint** de listado. **No** crear `SearchController`, ruta `/search` ni contrato HTTP paralelo.

```http
GET /delivery-history/shipments?q=&courier=&deliveryType=&milestone=&limit=&offset=
```

| Capa | Decisión |
|------|----------|
| HTTP | Solo `DeliveryHistoryController` → `GET /delivery-history/shipments` |
| Application | `ListShipmentsUseCase` orquesta; puede delegar a `SearchShipmentsUseCase` cuando hay criterios de búsqueda/filtro |
| Infrastructure | `DeliveryHistorySearchRepository` (lectura índice) + `DeliveryHistoryViewRepository` (list/hydrate) — repos dedicados **internos**, no expuestos como API |
| Escritura índice | Siempre en tx de proyección (`packageNewMilestoneProjection`) — **nunca** desde use case de list |

**Lectura — flujo unificado con y sin `q`:**

| Escenario | Query plan |
|-----------|------------|
| Sin `q`, sin filtros | `WHERE is_listable ORDER BY updated_at DESC` (hoy) |
| Sin `q`, con filtros | Filtros sobre `delivery_history_views` + partial indexes |
| Con `q` (válido, ver guardrails) | Prefix scan: `field_value LIKE normalize(q) \|\| '%'` en `search_index` → join/intersect en `delivery_history_views` |
| Con `q` + filtros | Intersección: matches del índice ∩ `courier` / `deliveryType` / `current_milestone` |

**Semántica de respuesta paginada (con y sin `q`):**

- `total` = cardinalidad del **conjunto filtrado** (prefix + filtros + `is_listable`), no el total global de views.
- `results` = página `limit`/`offset` sobre ese conjunto, `ORDER BY updated_at DESC`.
- La primera página puede devolver **menos filas que `limit`** si hay pocos matches — comportamiento esperado.
- Cambiar `q` o un filtro **reinicia** la paginación en el cliente (`offset = 0` / `page = 1`).
- Sin `q` y sin filtros: listado reciente listable (comportamiento actual de grilla).

**Query params (listado):**

| Param | Tipo | Reglas |
|-------|------|--------|
| `q` | `string` opcional | Prefix search; ver guardrails § longitud mínima |
| `courier` | código catálogo | Exact match (`ANDREANI`, `ENVIOPACK`); omitir si no filtra |
| `deliveryType` | código catálogo | Exact match (`HD`, `SPU`); uppercase |
| `milestone` | `MilestoneKey` | Exact match en `current_milestone`; validar contra catálogo Block 12 |
| `limit` | `1..100` | Default `20` |
| `offset` | `≥ 0` | Default `0` |

**Guardrails backend (protección DB — obligatorios en C.6):**

| Regla | Valor MVP | Comportamiento |
|-------|-----------|----------------|
| Longitud mínima `q` | **3** caracteres tras `normalizeSearchToken` | Si `q` está presente y normalizado tiene longitud `< 3` → **400** `Invalid search query` (no ejecutar prefix scan) |
| Longitud máxima `q` | **64** | Truncar o **400** — preferir **400** si excede |
| `q` vacío / solo whitespace | — | Omitir búsqueda; equivaler a list sin `q` |
| Normalización | Igual que escritura índice | `trim` + `uppercase`; tracking numérico sin espacios internos |
| Match SQL | `LIKE prefix%` | **No** `ILIKE`, **no** `%prefix%` (rompe `varchar_pattern_ops`) |
| Con `q` válido | JOIN en una query | Preferir SQL raw con subquery/join; evitar materializar miles de `view_id` en app |
| Sin `q` | No tocar `search_index` | Prisma sobre `delivery_history_views` |
| `limit` máximo | `100` | Ya en DTO; no elevar sin revisión |

**Contrato frontend — `backoffice-shipments-zone` / SXP (wiring con `ShipmentList`):**

Spec dedicada: **`backoffice-list-search-integration.md`**. El DS (`@fravega-it/logistic-ds`) ya expone barra de búsqueda y filtros vía `filters` + `onFiltersChange`; el cableado a API es responsabilidad del zone (backend C.5/C.6 ✅).

| Práctica | Regla MVP | Motivo |
|----------|-----------|--------|
| No enviar `q` hasta longitud mínima | **≥ 3** caracteres (tras trim) | Alineado con guardrail backend; evita prefix scans de 1–2 chars (`E`, `EP`) |
| Debounce en texto de búsqueda | **300 ms** | Reduce ráfagas de requests mientras el usuario escribe |
| Filtros discretos (`courier`, `deliveryType`, `milestone`) | Refetch inmediato | Sin debounce |
| Paginación | `mode: 'server'` | Ya en SXP-209; **obligatorio** con search server-side |
| `onFiltersChange` | Actualizar state + `page = 1` + refetch | Hoy es no-op; debe cablearse |
| Mapeo DS → API | `filters.query` → `q`; `filters.status` → `milestone` | El DS usa `status`; el API usa `MilestoneKey` |
| Valores de filtro | **Códigos** de catálogo (`ANDREANI`, `HD`, `WAREHOUSE_RECEIVED`) | Labels solo en columnas; `GET /catalog/*` para dropdowns |
| `searchFields` del DS | **No** usar filtrado client-side cuando hay `q` server | Con paginación server filtra solo la página cargada (~10 filas) — UX incorrecta |
| React Query `queryKey` | Incluir `q`, filtros, `limit`, `offset` | Invalidar cache al cambiar criterios |
| Request sin búsqueda activa | Solo `limit` + `offset` (+ filtros si aplica) | No mandar `q=` vacío |

**Repositorio (projector — lectura):**

- `DeliveryHistorySearchRepository` (recomendado) o métodos en `DeliveryHistoryViewRepository`:
  - `searchListable(criteria)` → `{ results, total }` — único entry de lectura para el use case
- Con `q`: raw SQL con JOIN/subquery preferido; sin `q`: Prisma `findMany` + `count`
- Mapper de list: reutilizar `DeliveryHistoryViewMapper.toDomain` en filas devueltas — evitar N+1

**Performance:**

- Prefix search usa `varchar_pattern_ops` (RFC2)
- Sin `q`: no tocar `search_index`
- Paginación: `LIMIT`/`OFFSET` (o equivalente) **después** de filtrar; `COUNT(*)` con los mismos predicados que `results`
- Particionado de tablas operativas **no** en este block (Block 13)

**Validación:**

- Unit: normalizador + mapper index rows desde bootstrap OE_LAUNCH; rechazo `q` &lt; 3 chars
- Integration: insert view + index → prefix `EP0129` devuelve view correcta; `total` coherente con filtros
- E2E orchestrator (opcional): asserts SQL en índice + `GET ...?q=<prefix EP>` — hoy cubierto por `warehouse` + Artillery `search-load`
- Sin orden comercial: búsqueda por `PACKAGE_EXTERNAL_ID` y `TRACKING` funciona; `ORDER_ID` ausente hasta reconciliar

---

### Block 12: Catálogos de Filtros & Endpoints de Facetas

**Decisión:** Dropdowns desde tablas catálogo seeded. `subStatus` nunca es filtro.

**Catálogo `MilestoneKey` v1 (DEFINITION CLOSED — HTTP ✅ C.2 partial):**

| `code` | `label` (es) | `sort_order` |
|--------|--------------|--------------|
| `WAREHOUSE_RECEIVED` | Recibido en depósito | 10 |
| `LAST_MILE_DISPATCHED` | Despachado a última milla | 20 |
| `DELIVERED` | Entregado | 30 |
| `FAILED` | Entrega fallida | 40 |
| `CANCELLED` | Cancelado | 50 |

**Qué existe:** logistics operators + operation types; `current_milestone` ✅; **`GET /delivery-history/catalog/milestones`** + mock (`MOCKS_ENABLED`); incluido en `GET /catalog/all`.

**Qué falta:**
- Tabla `milestones` + seeder DB (C.2 persistencia)
- Validación query `milestone` en list contra catálogo (opcional endurecimiento)
- Wiring FE: `onFiltersChange` → API — spec **`backoffice-list-search-integration.md`**

---

### 🔒 Block 13: Particionado & TTL de Search (DIFERIDO — post migration inicial)

**Regla crítica:** **No particionar** hasta cerrar la migration inicial única y estabilizar el modelo. Las tablas pueden evolucionar antes de salida a producción; evitar decenas de migrations intermedias.

**Cuándo ejecutar:** Después de que la migration inicial full esté mergeada y el modelo de lectura/escritura validado E2E (Blocks 6–12).

**Candidatos a particionado (RFC2 §8):**

| Tabla | Estrategia | Motivo |
|-------|------------|--------|
| `inbox_events` | RANGE `created_at` mensual | Cola transitoria; detach/drop de particiones viejas |
| `outbox_events` | RANGE `created_at` mensual | Idem |
| `delivery_history_search_index` | RANGE `updated_at` o heredar TTL vía views | Evitar resultados de envíos > N meses en búsqueda activa |
| `package_events` | Evaluar post-MVP | Append-only de alto volumen |

**Search TTL (diseño futuro):**
- Búsqueda activa solo sobre envíos con `updated_at` (o `created_at`) dentro de ventana configurable (ej. 90 días).
- Particiones viejas de search_index: detach + archive/drop (mismo procedimiento RFC2 §8.3).
- Listado sin `q` puede seguir mostrando recientes vía `delivery_history_views` con misma ventana, o criterio de negocio separado.

**No hacer ahora:**
- `CREATE TABLE ... PARTITION BY` en schema Prisma hasta Block 13
- Procedures `create_monthly_partitions` / `detach_old_partitions` (documentados en RFC2, implementar en Block 13)

---

### 🔄 Block R1: Refactor — Semántica de Estado en Listado (post Block 2)

**Motivo:** Block 2 cerró `status` + `subStatus` en `ListShipmentsResultItem`. La decisión de producto (filtros + grilla) usa **milestone** como estado visible. Este block evita reabrir Block 2 retroactivamente para **lista**.

**Cambios propuestos:**
- `ListShipmentsResultItem`: agregar `milestone: MilestoneKey`; deprecar `status`/`subStatus` en listado (o mapear `status` → `milestone` con alias temporal para frontend)
- `ListShipmentsResultItem.id`: garantizar UUID de `DeliveryHistoryView` (navegación a detalle)
- Mocks (`deliveryHistoryMocks.registry`): alinear filas a `milestone`
- Controller decorator mock: soportar query `q`, `courier`, `deliveryType`, `milestone` en modo mock (filtrado en memoria sobre mock data)
- **No** nuevo controller HTTP — mismos query params en `GET /delivery-history/shipments` (Block 11)

**Frontend (SXP / `backoffice-shipments-zone`):** ver Block 11 § contrato frontend — `filters.query` → `q`, debounce 300 ms, mínimo 3 caracteres, paginación server, catálogos para códigos de filtro.

**Fuera de scope R1:** evolución de `GetShipmentDetailOutputDto` → ver **Block R2**.

**Compatibilidad:** Si frontend aún consume `status`, mantener ambos campos una release; documentar removal en Block 10.

---

### 🔄 Block R2: Refactor — Contrato de Detalle v2 (post Block 8 §8.1)

**Motivo:** Gap entre `GetShipmentDetailOutputDto` v1 y los datos que necesita `ShipmentDetailViewModel` en backoffice. Spec cerrada 2026-06-25: **contrato de datos normalizados**, no isomórfico al DS. El trabajo es modificar DTO + use case (+ mocks generados), no el modelo de dominio.

**Alcance:**
- Extender `GetShipmentDetailOutputSchema` según Block 8 §8.1 (revisión 2026-06-25)
- Regla serialización: keys del contrato siempre presentes; valor `null` si dato/enriquecimiento no disponible
- **Capa A (operativo):** v1 + `summary.totalVolumeCubicCm`, `packages[].volumeCubicCm`, `statusItems` opcional, corregir `totalWeightGrams` (`null` si algún bulto sin peso)
- **Capa B (comercial):** `logistics` semántico + `shipmentInfo` plano — schema con `null` hasta ingesta/proyección (no bloquear merge de A)
- **Fuera de scope API:** `quickActions`, URLs tracking, labels/colores/emojis, `LogisticsBlockDto` con `fields[].label`
- Regenerar mocks: `npm run mocks:generate:delivery-history` (no editar registry a mano); subset de demos con comercial sintético permitido solo en generador
- Actualizar Swagger + tests unitarios (shape estable, comercial `null` sin fuente)

**No hacer en R2:**
- Reinterpretar `canonicalStatus` / `MilestoneKey` (API ya normaliza)
- Agregar agregación de estado entre packages
- Fabricar `logistics` con datos inventados en el use case real
- Exponer catálogo de URLs de carrier desde projector

**Archivos esperados:**
- `src/modules/deliveryHistory/dtos/getShipmentDetail.dto.ts`
- `src/modules/deliveryHistory/useCases/getShipmentDetail.useCase.ts` (OutputDto vía métodos privados; sin `*.response.mapper.ts`)
- `src/modules/deliveryHistory/repositories/deliveryHistoryView/deliveryHistoryView.repository.mapper.ts` (`toDetail`, read model)
- `src/infrastructure/database/seeders/generateDeliveryHistoryMocks.ts`
- `src/modules/deliveryHistory/swagger/deliveryHistory.swagger.ts`
- `test/unit/deliveryHistory/getShipmentDetail.useCase.spec.ts` (nuevo o extendido)

**Frontend (zone, fuera de R2 projector pero dependiente del contrato):**
- Borrar `mockModel` inline en `backoffice-shipments-zone/.../[id]/page.tsx`
- `mapDetailToViewModel(dto)` + `detailView.config.ts` (presentación) + config local `courier` → tracking URL

**Backlog API/sync (post-R2, para poblar Capa B):**

| Campo DTO | Fuente futura |
|-----------|---------------|
| `logistics.destination.*` | Order / Delivery reconciliado (nodo comercial — `logistics.md`) |
| `logistics.origin.*` | CD / WMS metadata |
| `logistics.return.*` | Seller / SPU |
| `shipmentInfo.journeyId` | Journey proyectado |
| `shipmentInfo.routeId` | Dispatch WMS |
| `shipmentInfo.sortable` | Metadata WMS |
| `items[].weightGrams/volumeCubicCm` | Catálogo comercial |

Gancho existente: `deliveryLink` en gates API (`consumeOeLaunch`, dispatch, Andreani) — hoy no proyecta direcciones.

**Dependencias:** Block 11 opcional para lookup `displayId`; R2 puede shippear con UUID-only.

**Status:** **DONE** (implementación 2026-06-25). Validar local: `MOCKS_ENABLED=true` → `GET /delivery-history/shipments` → `GET /delivery-history/shipments/:id`.

---

## Execution Priority

**Actualizado 2026-06-25.** Status detallado: `plan.md`. Onboarding + handoff search: `prompt.md`.

1. **SP.1** — search performance P1 (`search-performance-spec.md`) — query única prefix, índices parciales, select mínimo sin `q`
2. **C.4** — `PackageEvent.shipmentId` = active trail en proyección
3. **C.2** persistencia catálogo `milestones` (HTTP ✅)
4. Block 13 particionado search/inbox (post-MVP)

**Completado recientemente:** **C.5 + C.6 search** · B.7.2 bulk relay · B.6 orchestrator E2E · Block R2 detalle v2 · C.7 terminales 5 keys.

---

## Decisions Already Made

| Decision | State |
|----------|-------|
| Package = unit of traceability | FINAL |
| Shipment ≠ DeliveryView | FINAL |
| InboxEvent as normalized transport | FINAL |
| PackageMilestones as snapshot | FINAL |
| DeliveryHistoryView as UI aggregate | FINAL |
| API→Projector con payload de cambios (no signal+pull) | FINAL |
| Mocks via decorator toggle | FINAL |
| Listable post primer gate listable (MVP: `WAREHOUSE_RECEIVED`); pre-gate cancel no en grilla | FINAL — `plan.md` § B.8 |
| Trazabilidad y estado por bulto (sin agregación cross-package) | FINAL |
| Block 9 (external cases) via flag + fallback | PROPOSED |
| Search via `delivery_history_search_index` (PostgreSQL prefix) | FINAL |
| Search HTTP: param `q` en `GET /delivery-history/shipments` (sin controller/ruta aparte) | FINAL — 2026-06-25 |
| Longitud mínima `q`: 3 chars (FE no envía antes; BE 400 si viola) | FINAL — MVP |
| List paginado: `total` = conjunto filtrado; `results` = página sobre ese conjunto | FINAL |
| Catálogos filtro en DB: `logistics_operators`, `operation_types` | FINAL |
| Milestone = estado UI (`MilestoneKey`); subStatus nunca en filtros | FINAL — ver `domain-model.md` §4 |
| Tramos binarios `FIRST_MILE` / `LAST_MILE`; warehouse dentro de última milla | FINAL — ver `domain-model.md` §3 |
| Catálogo `MilestoneKey` (5 valores v1) | FINAL — Block 12 HTTP ✅; persistencia C.2 pending |
| Particionado inbox/outbox/search — post migration inicial | FINAL |
| `CUSTOMER_DNI` en search index cuando exista reconciliación comercial | PROPOSED |
| Detalle HTTP: datos normalizados, shape estable, comercial nullable (`Block 8` §8.1 rev. 2026-06-25) | FINAL |
| Detalle HTTP: sin `quickActions` ni presentación UI (labels/colores/URLs) — adapter frontend | FINAL |
| Lista navega con `DeliveryHistoryView.id` (UUID), no `displayId` | FINAL |

---

## Known Constraints

- Migration `current_milestone` ✅ C.1; lógica terminales 5 keys ✅ **C.7**
- Particionado explícitamente **fuera de scope** hasta post-MVP (Block 13)
- **Search funcional ✅ C.5 + C.6:** índice en tx de proyección; list `q` + filtros en `GET /delivery-history/shipments`; mocks decorator alineado
- Search **performance** pendiente — `search-performance-spec.md` (SP.1); denormalización/particionado → P2 / Block 13
- Catálogos HTTP: `logistics_operators`, `operation_types`, milestones (static/mock)
- Spec search cerrada (HTTP `q` unificado, guardrails FE/BE, paginación filtrada) — Block 11; integración FE → `backoffice-list-search-integration.md`
- API → projector E2E no validado en entorno integrado (replay outbox pendiente — Block 10)
- Mocks static from `__mocks__/deliveryHistory/*.json` (lint ruidoso en registry generado)
- Frontend: detalle v2 (**Block R2** ✅) + list search spec; wiring FE pendiente
- Warehouse-centric schema ready; external-only cases need validation (Block 9)
