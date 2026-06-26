# Contrato detalle API/Backoffice (`ShipmentDetailViewModel`)

Integración `backoffice-shipments-zone` a `shipments-history-projector`.

**Fuente de verdad del schema:** `shipments-history-projector/src/modules/deliveryHistory/dtos/getShipmentDetail.dto.ts`  
**Mock UI de referencia:** `backoffice-shipments-zone/src/app/(authenticated)/(home)/[id]/page.tsx`

---

## 1. Flujo lista - detalle

| Paso | Acción |
|------|--------|
| 1 | `GET /api/v1/delivery-history/shipments?limit=&offset=` (+ opcionales `q`, `courier`, `deliveryType`, `milestone` — ver [1.1](#11-búsqueda-y-filtros-listado)) |
| 2 | Tomar `results[].id` (**UUID** de `DeliveryHistoryView`) — **no** `order`, **no** `shipment` |
| 3 | `GET /api/v1/delivery-history/shipments/{id}` |

### 1.1 Búsqueda y filtros (listado)

**Decisión:** un solo endpoint. **No** existe `GET /delivery-history/search`. El DS (`ShipmentList`) cablea `filters.query` → param `q`.

```http
GET /api/v1/delivery-history/shipments?q=&courier=&deliveryType=&milestone=&limit=&offset=
```

| Param API | Origen UI (`ShipmentList`) | Reglas |
|-----------|----------------------------|--------|
| `q` | `filters.query` | Prefix search server-side; **no enviar** hasta ≥ **3** caracteres (trim); debounce **300 ms** |
| `courier` | `filters.courier` | Código catálogo (`ANDREANI`); omitir si `ALL` |
| `deliveryType` | `filters.deliveryType` | Código (`HD`, `SPU`); omitir si `ALL` |
| `milestone` | `filters.status` | `MilestoneKey`; omitir si `ALL`; labels vía `GET /catalog/milestones` |
| `limit` / `offset` | paginación server | `offset = (page - 1) * pageSize`; reset `page = 1` al cambiar filtros |

**Respuesta paginada:**

- `total` = cantidad de envíos que matchean criterios (no el universo global).
- `results` = slice de esa cantidad según `limit`/`offset`.
- No usar `searchFields` del DS para filtrar en cliente cuando la paginación es server — dispara `q` al API.

**Referencia implementación FE:** rama `feature/SXP-209-integracion-listado-envios-backend` — extender `shipments.api.ts` / `useShipments` (hoy solo `limit`/`offset`).

Spec completa (guardrails BE, arquitectura projector): `blocks-projector.md` Block 11.

### Lista (`ListShipmentsResultItem`)

| Campo API | Tipo | Nullable | Uso en UI |
|-----------|------|----------|-----------|
| `id` | `uuid` | no | Navegación al detalle |
| `order` | `string` | **si** | Columna “Orden” (`128-18867800-00567429`) |
| `shipment` | `string` | no | Columna “Envío” — `displayId` visible (ej. `EP000000001N-01`) |
| `parcel` | `string` | **si** | Bulto primario |
| `trackingId` | `string` | **si** | Tracking visible |
| `courier` | `string` | **si** | Código carrier |
| `deliveryType` | `string` | **si** | Código (`SPU`, `HD`, …) — label vía catálogo *(detalle en [8](#8-catálogos))* |
| `status` | `string` | no | Macro estado (grilla legacy; milestone en `currentMilestone`) |
| `subStatus` | `string` | **si** | Diagnóstico |
| `currentMilestone` | `string` | **si** | Estado milestone en grilla |
| `etd` | `datetime` ISO | **si** | ETD |

---

## 2. Principio: datos vs presentación

La API devuelve **datos normalizados**. El `ShipmentDetailViewModel` de `@fravega-it/logistic-ds` agrega capa de presentación que **no** viene en el HTTP:

| Responsable | Ejemplos |
|-------------|----------|
| **API (projector)** | `status`, `courier`, `logistics.destination.recipientName`, `summary.totalWeightGrams` |
| **Frontend (zone)** | `title`, `logistics[].color`, `fields[].label`, `quickActions`, `"3.2 kg"`, fechas formateadas, `onBack` |

El mock inline en `page.tsx` mezcla ambas capas. Al integrar, reemplazar `mockModel` por `mapDetailToViewModel(apiResponse)` + config local (`detailView.config.ts`).

**Fuera del contrato HTTP:** `quickActions`, URLs de seguimiento por carrier, emojis, colores, `emphasized`, `tone`, `italic`.

---

## 3. Regla de shape estable

Toda **key definida en el schema v2** aparece **siempre** en el JSON:

- Valor ausente → `null` (escalares/objetos) o `[]` (arrays).
- **Nunca** se omiten keys del contrato.
- Sin enriquecimiento comercial → `logistics.origin|destination|return` pueden ser `null` (el objeto `logistics` **sí** está presente con sus tres hijos).

Ejemplo sin datos comerciales (caso operativo típico hoy):

```json
{
  "logistics": {
    "origin": null,
    "destination": null,
    "return": null
  },
  "shipmentInfo": {
    "journeyId": null,
    "sortable": null,
    "routeId": null,
    "committedDeliveryAt": null,
    "rescheduledDeliveryAt": null
  }
}
```

---

## 4. DTO completo — `GetShipmentDetailOutputDto`

Leyenda: **R** = requerido (no null) · **N** = nullable · **A** = array (siempre presente, puede ser `[]`)

### 4.1 Raíz

| Campo | Tipo | Null | Descripción |
|-------|------|------|-------------|
| `id` | `uuid` | R | UUID canónico (`DeliveryHistoryView.id`). Mismo valor que en lista. |
| `shipmentId` | `string` | R | Identificador visible (`displayId`). Ej. `EP000000001N-01`. **No** usar en la URL del GET. |
| `orderId` | `string` | N | Orden comercial (`128-…`) si reconciliada. |
| `externalOrderId` | `string` | N | Referencia externa de orden (suele coincidir con `orderId` cuando existe). |
| `trackingId` | `string` | N | Tracking principal visible. |
| `courier` | `string` | N | Código carrier (`ANDREANI`, `ENVIOPACK`, …). |
| `deliveryType` | `string` | N | Código tipo entrega. Label → `GET /delivery-history/catalog/operation-types` *(10)*. |
| `status` | `string` | R | `canonicalStatus` del envío/vista (ej. `IN_TRANSIT`, `DELIVERED`). |
| `subStatus` | `string` | N | Diagnóstico carrier/operativo. |
| `etd` | `datetime` ISO | N | Estimated delivery. |
| `lastOccurredAt` | `datetime` ISO | N | Último evento relevante. |
| `summary` | `object` | R | Ver 4.2. Siempre presente. |
| `statusItems` | `array` | A | Ver 4.3. Espejo por bulto; puede derivarse de `packages[]`. |
| `logistics` | `object` | R | Ver 4.4. Siempre presente; hijos `null` sin comercial. |
| `shipmentInfo` | `object` | R | Ver 4.5. Siempre presente; campos internos `null` sin fuente. |
| `segments` | `array` | A | Tramos logísticos (`FIRST_MILE` / `LAST_MILE`). |
| `packages` | `array` | A | Bultos con timeline e ítems. |

### 4.2 `summary`

| Campo | Tipo | Null |
|-------|------|------|
| `totalPackages` | `number` | R |
| `totalItems` | `number` | R |
| `totalWeightGrams` | `number` | N — `null` si algún bulto sin peso |
| `totalVolumeCubicCm` | `number` | N — `null` si algún bulto sin volumen |

### 4.3 `statusItems[]`

| Campo | Tipo | Null |
|-------|------|------|
| `packageExternalId` | `string` | R |
| `status` | `string` | R |
| `subStatus` | `string` | N |

### 4.4 `logistics` (siempre las tres keys)

| Campo | Tipo | Null |
|-------|------|------|
| `origin` | `LogisticsOrigin` | N — bloque entero `null` sin CD/origen |
| `destination` | `LogisticsDestination` | N |
| `return` | `LogisticsReturn` | N |

**`LogisticsOrigin`** (si `origin` no es `null`):

| Campo | Tipo | Null |
|-------|------|------|
| `distributionCenterCode` | `string` | N |
| `distributionCenterName` | `string` | N |
| `address` | `string` | N — dirección ya normalizada por la fuente; API no descompone |
| `addressComment` | `string` | N — aclaración de entrega/ubicación cuando la fuente la provee |

**`LogisticsDestination`**:

| Campo | Tipo | Null |
|-------|------|------|
| `recipientName` | `string` | N |
| `address` | `string` | N |
| `addressComment` | `string` | N |

**`LogisticsReturn`**:

| Campo | Tipo | Null |
|-------|------|------|
| `sellerName` | `string` | N |
| `address` | `string` | N |
| `addressComment` | `string` | N |
| `reference` | `string` | N |

### 4.5 `shipmentInfo`

| Campo | Tipo | Null |
|-------|------|------|
| `journeyId` | `string` | N |
| `sortable` | `boolean` | N |
| `routeId` | `string` | N |
| `committedDeliveryAt` | `datetime` ISO | N |
| `rescheduledDeliveryAt` | `datetime` ISO | N |

No duplicar en `shipmentInfo` lo que ya está en raíz: `orderId`, `externalOrderId`, `deliveryType`, `courier`, `trackingId`, `etd`, `lastOccurredAt`.

### 4.6 `segments[]` - Para vista mas granular en timelines, no es necesario en una v1

| Campo | Tipo | Null |
|-------|------|------|
| `id` | `uuid` | R |
| `sourceSystem` | `string` | R |
| `trailType` | `string` | R — `FIRST_MILE` / `LAST_MILE` |
| `carrierCode` | `string` | N |
| `trackingNumber` | `string` | N |
| `status` | `string` | R |
| `subStatus` | `string` | N |
| `shippedAt` | `datetime` ISO | N |
| `deliveredAt` | `datetime` ISO | N |
| `estimatedDelivery` | `datetime` ISO | N |

### 4.7 `packages[]` - Dado el modelo, siempre va a ser 1, porque es trazabilidad POR BULTO.

| Campo | Tipo | Null |
|-------|------|------|
| `id` | `uuid` | R |
| `externalId` | `string` | R — identificador bulto (EP…) |
| `trackingId` | `string` | N |
| `courier` | `string` | N |
| `status` | `string` | R |
| `subStatus` | `string` | N |
| `weightGrams` | `number` | N |
| `volumeCubicCm` | `number` | N |
| `milestones` | `object` | R — 10 timestamps `*At`, cada uno **N** |
| `items` | `array` | A |
| `timeline` | `array` | A |

**`milestones`:** `readyToCollectAt`, `collectedAt`, `journeyAnnouncedAt`, `oeLaunchAt`, `classifiedAt`, `dispatchedAt`, `pickedUpByCarrierAt`, `consolidatedAtCarrierAt`, `inTransitAt`, `deliveredAt` — todos `datetime` ISO **nullable**.

### 4.8 `packages[].items[]` - 1 o n

| Campo | Tipo | Null |
|-------|------|------|
| `id` | `uuid` | R |
| `sku` | `string` | R |
| `name` | `string` | N |
| `quantity` | `number` | R |
| `weightGrams` | `number` | N — comercial; `null` sin enriquecimiento de producto |
| `volumeCubicCm` | `number` | N |

### 4.9 `packages[].timeline[]`

| Campo | Tipo | Null |
|-------|------|------|
| `id` | `uuid` | R |
| `eventName` | `string` | R |
| `status` | `string` | N |
| `subStatus` | `string` | N |
| `location` | `string` | N |
| `actor` | `string` | N |
| `description` | `string` | N |
| `occurredAt` | `datetime` ISO | R |
| `deliveryFailureContext` | `object` | N — ver abajo |

**`deliveryFailureContext`** (cuando no es `null`):

| Campo | Tipo | Null |
|-------|------|------|
| `reasonCode` | `string` | R |
| `reasonLabel` | `string` | N |
| `comment` | `string` | N |

---

## 5. Mapeo API → `ShipmentDetailViewModel`

Implementar en zone como `mapDetailToViewModel(dto, config)`.

### 5.1 Cabecera y navegación

| ViewModel | Fuente API | Notas |
|-----------|------------|-------|
| `title` | — | Copy fijo / i18n en zone |
| `shipmentId` | `shipmentId` | Visible en breadcrumb (no confundir con `id` UUID) |
| `onBack` | — | `router.push('/')` |

### 5.2 `logistics[]` (bloques DS)

Config en zone: `key`, `title`, `color` por bloque (`origen` / `destino` / `devolucion`).

| Bloque DS | API | Campos DS ← API |
|-----------|-----|-----------------|
| Origen | `logistics.origin` | `centro` ← `distributionCenterCode` + `distributionCenterName`; `direccion` ← `address`; comentario opcional ← `addressComment` |
| Destino | `logistics.destination` | `cliente` ← `recipientName`; `direccion` ← `address`; comentario opcional ← `addressComment` |
| Devolución | `logistics.return` | `seller` ← `sellerName`; `direccion` ← `address`; comentario opcional ← `addressComment`; `referencia` ← `reference` |

Si `logistics.origin === null` → no renderizar bloque origen (o bloque vacío según UX).

### 5.3 `shipmentInfo` (columnas y summary DS)

| Campo DS (mock) | Fuente API |
|-----------------|------------|
| `orden` | `orderId` ?? `externalOrderId` |
| `tipoEntrega` | `deliveryType` → label catálogo operation types |
| `jornada` | `shipmentInfo.journeyId` |
| `entregaComprometida` | `shipmentInfo.committedDeliveryAt` ?? `etd` |
| `sorteable` | `shipmentInfo.sortable` → `"Sí"` / `"No"` / ocultar si `null` |
| `ultimaActualizacion` | `lastOccurredAt` → formatear |
| `refExterna` | `externalOrderId` ?? `shipmentId` |
| `courier` | `courier` → label catálogo logistics operators |
| `ruta` | `shipmentInfo.routeId` |
| `entregaReplanificada` | `shipmentInfo.rescheduledDeliveryAt` |
| `trackingEnvio` | `trackingId` |
| `totalBultos` | `summary.totalPackages` |
| `totalItems` | `summary.totalItems` |
| `volumen` | `summary.totalVolumeCubicCm` → formatear m³ |
| `peso` | `summary.totalWeightGrams` → formatear kg |

Labels (`"Orden"`, `"Tipo de Entrega"`, …) y `tone` del summary → **config zone**, no API.

### 5.4 `statusItems[]`

| ViewModel | API |
|-----------|-----|
| `key` | generar en FE (`st-${index}`) |
| `label` | `statusItems[].packageExternalId` |
| `status` | `statusItems[].status` → label humano si aplica |
| `subtitle` | `statusItems[].subStatus` |

Alternativa: derivar desde `packages[]` sin usar `statusItems`.

### 5.5 `packages[]`

| ViewModel | API |
|-----------|-----|
| `id` | `packages[].externalId` (visible) o `packages[].id` (uuid interno) |
| `courier` | `packages[].courier` → label |
| `tracking` | `packages[].trackingId` |
| `status` | `packages[].status` → label |
| `statusNote` | `packages[].subStatus` |
| `defaultExpanded` / `collapsible` / `timelineShowToggle` | defaults UX en zone |

**`items[]`:**

| ViewModel | API |
|-----------|-----|
| `key` | generar en FE |
| `name` | `items[].name` |
| `sku` | `items[].sku` |
| `qty` | `items[].quantity` |
| `weight` | `items[].weightGrams` → formatear |
| `details` | `items[].volumeCubicCm` → `"Vol: X m³"` |

**`timeline[]`:**

| ViewModel | API |
|-----------|-----|
| `key` | generar en FE |
| `title` | derivar de `eventName` / `status` (mapeo en zone) |
| `status` | `timeline[].subStatus` ?? `timeline[].status` |
| `date` | `timeline[].occurredAt` → formatear |
| `highlighted` | regla UX (ej. último evento) |
| `place` | `timeline[].location` |
| `description` | `timeline[].description`; anexar `deliveryFailureContext` si existe |

### 5.6 `quickActions` — solo frontend

| ViewModel | Fuente |
|-----------|--------|
| `quickActions` | `courier` + `trackingId` + mapa local de URLs por carrier |

No existe en el DTO de detalle.

---

## 6. Ejemplo mínimo operativo (sin comercial)

Respuesta típica con `MOCKS_ENABLED=false` antes de enriquecimiento comercial:

```json
{
  "id": "3aabbdc1-473e-4dbb-93ea-ac9ef6e8e6c4",
  "shipmentId": "EP000000003N-01",
  "orderId": null,
  "externalOrderId": null,
  "trackingId": "360002990600003",
  "courier": "ANDREANI",
  "deliveryType": "STANDARD",
  "status": "IN_TRANSIT",
  "subStatus": null,
  "etd": "2026-01-05T01:00:00.000Z",
  "lastOccurredAt": "2026-01-02T11:00:00.000Z",
  "summary": {
    "totalPackages": 1,
    "totalItems": 2,
    "totalWeightGrams": 950,
    "totalVolumeCubicCm": null
  },
  "statusItems": [
    {
      "packageExternalId": "EP000000003N-01",
      "status": "IN_TRANSIT",
      "subStatus": null
    }
  ],
  "logistics": {
    "origin": null,
    "destination": null,
    "return": null
  },
  "shipmentInfo": {
    "journeyId": null,
    "sortable": null,
    "routeId": null,
    "committedDeliveryAt": null,
    "rescheduledDeliveryAt": null
  },
  "segments": [],
  "packages": []
}
```

La UI debe renderizar sin error: bloques logísticos ocultos o vacíos, campos de `shipmentInfo` omitidos cuando llegan `null`, timeline/packages desde datos operativos cuando existan.

---

## 7. Mocks

~20% de filas mock incluyen `logistics` y `shipmentInfo` poblados (demo comercial). El resto ejercita el caso `null`.

---

## 8. Catálogos

Los catálogos HTTP traducen **códigos** que vienen en lista/detalle (`courier`, `deliveryType`, `currentMilestone`) a **labels** para dropdowns y para el adapter de detalle. La API de envíos **no** embebe labels de presentación en el DTO de detalle; el frontend resuelve `code → label` con estos endpoints.

**Base URL (projector):** `GET /api/v1/delivery-history/catalog/...`

### 8.1 Formato común

Cada entrada es `{ value, label }`:

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `value` | `string` | Código estable (el que llega en list/detail) |
| `label` | `string` | Texto para UI (español en seeds actuales) |

Respuesta por dimensión (`CatalogDimensionOutputDto`):

```json
{
  "items": [
    { "value": "ANDREANI", "label": "Andreani" },
    { "value": "ENVIOPACK", "label": "EnvioPack" }
  ]
}
```

### 8.2 Endpoints

| Endpoint | Uso en backoffice |
|----------|-------------------|
| `GET /delivery-history/catalog/logistics-operators` | Label de `courier` / `carrierCode`; filtros de grilla por operador |
| `GET /delivery-history/catalog/operation-types` | Label de `deliveryType` (ej. `HD` → `"Home Delivery"`) |
| `GET /delivery-history/catalog/milestones` | Label de `currentMilestone` en lista; filtro por milestone (`MilestoneKey`) |
| `GET /delivery-history/catalog/all` | Los tres catálogos en una sola respuesta |

**`GET /catalog/all`** — shape (`AllCatalogOutputDto`):

```json
{
  "logisticsOperators": [{ "value": "...", "label": "..." }],
  "operationTypes": [{ "value": "...", "label": "..." }],
  "milestones": [{ "value": "...", "label": "..." }]
}
```

Todas las keys anteriores **siempre** están presentes; cada una es un array (puede ser `[]` si no hay filas activas).

### 8.3 Valores de referencia (seeds actuales)

**Logistics operators** (`npm run db:seed:catalogs`):

| `value` | `label` |
|---------|---------|
| `ENVIOPACK` | EnvioPack |
| `ANDREANI` | Andreani |

**Operation types:**

| `value` | `label` |
|---------|---------|
| `HD` | Home Delivery |
| `SPU` | Store Pickup |

**Milestones** (`MilestoneKey` v1 — `domain-model.md` 4.1):

| `value` | `label` |
|---------|---------|
| `WAREHOUSE_RECEIVED` | Recibido en depósito |
| `LAST_MILE_DISPATCHED` | Despachado a última milla |
| `DELIVERED` | Entregado |
| `FAILED` | Entrega fallida |
| `CANCELLED` | Cancelado |

### 8.4 Uso en el adapter de detalle

| Código en DTO | Catálogo | Campo UI (mock) |
|---------------|----------|-----------------|
| `courier` | `logistics-operators` | `shipmentInfo.courier`, `packages[].courier` |
| `deliveryType` | `operation-types` | `shipmentInfo.tipoEntrega` |
| `currentMilestone` (lista) | `milestones` | columna estado milestone en grilla |


### 8.5 Persistencia y mocks

| Catálogo | Fuente hoy | Notas |
|----------|------------|-------|
| `logistics-operators` | Tabla `logistics_operators` (DB) | Requiere `db:seed:catalogs` |
| `operation-types` | Tabla `operation_types` (DB) | Idem |
| `milestones` | Seed estático en código | Tabla `milestones` pendiente en API - mock vía decorator |

Los catálogos **no** forman parte del body de detalle: conviene cargarlos una vez al montar la app (o cache por sesión) y reutilizar en lista + detalle.

### 8.6 Qué no son estos catálogos

- **No** incluyen URLs de tracking por carrier (`quickActions`) — config local del zone.
- **No** sustituyen el catálogo comercial de productos (peso/volumen por ítem); eso es enriquecimiento futuro en `packages[].items[]`.
- **No** traducen `status` / `subStatus` operativos del timeline; eso puede requerir mapeo aparte o mostrar el valor API.

---

## 9. Referencias

| Recurso | Ruta |
|---------|------|
| Schema Zod detalle | `shipments-history-projector/src/modules/deliveryHistory/dtos/getShipmentDetail.dto.ts` |
| Use case detalle (read model → OutputDto, métodos privados) | `shipments-history-projector/src/modules/deliveryHistory/useCases/getShipmentDetail.useCase.ts` |
| Read model detalle (Prisma → `DeliveryHistoryViewDetail`) | `shipments-history-projector/src/modules/deliveryHistory/repositories/deliveryHistoryView/deliveryHistoryView.repository.mapper.ts` |
| Schema Zod lista | `shipments-history-projector/src/modules/deliveryHistory/dtos/listShipments.dto.ts` |
| Schema Zod catálogos | `shipments-history-projector/src/modules/deliveryHistory/dtos/catalog.dto.ts` |
| Controller catálogos | `shipments-history-projector/src/modules/deliveryHistory/controllers/catalog.controller.ts` |
| OpenAPI detalle | `shipments-history-projector/src/modules/deliveryHistory/swagger/deliveryHistory.swagger.ts` |
| OpenAPI catálogos | `shipments-history-projector/src/modules/deliveryHistory/swagger/catalog.swagger.ts` |

