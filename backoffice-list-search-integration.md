# Integración backoffice — búsqueda y filtros en listado de envíos

Spec para el equipo de front (`backoffice-shipments-zone`) que integra la grilla de envíos con el motor de búsqueda del projector.

**Servicio:** `shipments-history-projector`  
**Base URL local:** `http://localhost:9200/api/v1`  
**Componente UI:** `ShipmentList` (`@fravega-it/logistic-ds`)

Contrato de lista/detalle general: `detail-api-frontend-contract.md`. Este documento cubre **solo search + filtros** del listado.

---

## 1. Decisión de integración

| Tema | Regla |
|------|-------|
| Endpoint | **Uno solo:** `GET /delivery-history/shipments` |
| Búsqueda | Query param `q` (prefix search server-side) |
| Filtros | Query params `courier`, `deliveryType`, `milestone` |
| Paginación | `limit` + `offset` sobre el **conjunto ya filtrado** |
| No existe | `GET /delivery-history/search` ni endpoint paralelo |

La UI del DS ya expone barra de búsqueda y dropdowns vía `filters` + `onFiltersChange`. El trabajo pendiente es **cablear state → refetch** con los query params de abajo.

---

## 2. Request

```http
GET /api/v1/delivery-history/shipments?q=&courier=&deliveryType=&milestone=&limit=&offset=
```

Todos los params de búsqueda/filtro son **opcionales**. Omitir el param cuando no aplica (no enviar `q=` vacío).

### 2.1 Parámetros

| Param | Tipo | Default | Descripción |
|-------|------|---------|-------------|
| `limit` | `number` | `20` | Máximo `100` |
| `offset` | `number` | `0` | Desplazamiento sobre resultados filtrados |
| `q` | `string` | — | Prefix search (ver §3) |
| `courier` | `string` | — | Código de operador logístico (exact match) |
| `deliveryType` | `string` | — | Código de tipo de operación (exact match, uppercase) |
| `milestone` | `MilestoneKey` | — | Filtro sobre `currentMilestone` (exact match) |

### 2.2 Valores válidos de `milestone`

| `milestone` | Label UI (es) |
|-------------|---------------|
| `WAREHOUSE_RECEIVED` | Recibido en depósito |
| `LAST_MILE_DISPATCHED` | Despachado a última milla |
| `DELIVERED` | Entregado |
| `FAILED` | Entrega fallida |
| `CANCELLED` | Cancelado |

Labels para dropdowns: `GET /delivery-history/catalog/milestones` (o `GET /delivery-history/catalog/all`).

### 2.3 Mapeo `ShipmentList` (DS) → API

| State del DS (`filters`) | Query API | Regla |
|--------------------------|-----------|--------|
| `query` | `q` | Ver guardrails §3 |
| `courier` | `courier` | Enviar **código** (`ANDREANI`, `ENVIOPACK`); omitir si `ALL` |
| `deliveryType` | `deliveryType` | Enviar **código** (`HD`, `SPU`); omitir si `ALL` |
| `status` | `milestone` | Enviar **MilestoneKey**; omitir si `ALL` |

**Display vs filtro:** las columnas pueden mostrar labels en español (adapter local), pero los filtros deben enviar **códigos** del catálogo. Catálogos:

| Filtro | Endpoint catálogo |
|--------|-------------------|
| `courier` | `GET /delivery-history/catalog/logistics-operators` |
| `deliveryType` | `GET /delivery-history/catalog/operation-types` |
| `milestone` | `GET /delivery-history/catalog/milestones` |

Shape catálogo: `{ items: [{ value, label }] }`.

### 2.4 Paginación

| Campo DS | Cálculo API |
|----------|-------------|
| `page` (1-based) | `offset = (page - 1) * pageSize` |
| `pageSize` | `limit` |

Usar `pagination.mode: 'server'` y `pagination.total` desde la respuesta API.

**Al cambiar `q` o cualquier filtro:** resetear `page = 1` antes del refetch.

---

## 3. Guardrails de búsqueda (`q`) — obligatorios en FE

| Regla | Valor | Motivo |
|-------|-------|--------|
| Longitud mínima antes de llamar API | **3** caracteres (tras `trim`) | Evita prefix scans demasiado amplios (`E`, `EP`) |
| Debounce en texto | **300 ms** | Reduce ráfagas mientras el usuario escribe |
| Filtros discretos | Refetch inmediato | Sin debounce en courier / deliveryType / milestone |
| No enviar `q` vacío | Omitir param | Equivalente a listado sin búsqueda |
| Longitud máxima | **64** caracteres | Límite API |

### Normalización esperada (informativo)

El backend normaliza `q` con `trim` + `uppercase`. El match es **prefix** (`LIKE 'PREFIX%'`). Ejemplos útiles en datos reales:

- `EP0` → envíos cuyo `displayId` empieza con `EP0…`
- `128` → órdenes Frávega `128-…`
- `360` → tracking Andreani numérico

### Errores

| Condición | HTTP | Body |
|-----------|------|------|
| `q` presente con &lt; 3 chars tras trim | `400` | `Invalid search query` |
| `q` &gt; 64 chars | `400` | `Invalid list parameters` o `Invalid search query` |
| `milestone` inválido | `400` | `Invalid list parameters` |
| `limit` / `offset` inválidos | `400` | `Invalid list parameters` |

**Implementación recomendada:** si `filters.query.trim().length` está entre 1 y 2, **no llamar al API** (mostrar listado anterior o vacío con hint UX). Solo disparar request cuando `length === 0` (sin `q`) o `length >= 3`.

---

## 4. Response

Sin cambios de shape respecto al listado existente. Cada fila:

```json
{
  "results": [
    {
      "id": "uuid-de-delivery-history-view",
      "order": "128-18867800-00567429",
      "shipment": "EP000000003N-01",
      "parcel": "EP000000003N-01",
      "trackingId": "360002990600003",
      "courier": "ANDREANI",
      "deliveryType": "HD",
      "status": "IN_TRANSIT",
      "subStatus": null,
      "currentMilestone": "LAST_MILE_DISPATCHED",
      "etd": "2026-01-05T01:00:00.000Z"
    }
  ],
  "total": 42
}
```

### Semántica de `total` y `results`

| Campo | Significado |
|-------|-------------|
| `total` | Cantidad de envíos que cumplen **todos** los criterios activos (`q` + filtros + listables) |
| `results` | Página actual (`limit`/`offset`) sobre ese conjunto, orden `updatedAt` DESC |

La primera página puede devolver **menos filas que `limit`** si hay pocos matches — es comportamiento correcto.

Navegación al detalle: usar siempre `results[].id` (UUID), nunca `shipment` / `order`.

---

## 5. Qué hace el backend al buscar

Prefix search sobre índice PostgreSQL en:

| Tipo indexado | Ejemplos de match con `q` |
|---------------|---------------------------|
| ID de bulto / envío visible | `EP000000003N-01`, variante sin guión |
| Orden comercial | `128-18867800-00567429` |
| Tracking | `360002990600003` |

Los filtros `courier`, `deliveryType`, `milestone` se aplican **además** de `q` (intersección).

---

## 6. Wiring recomendado (capa de servicios)

### 6.1 Extender el client HTTP

```typescript
export interface ShipmentsListParams {
  limit: number;
  offset: number;
  q?: string;
  courier?: string;
  deliveryType?: string;
  milestone?: string;
}

export function fetchShipments(params: ShipmentsListParams): Promise<ShipmentListApiResponse> {
  const search = new URLSearchParams({
    limit: String(params.limit),
    offset: String(params.offset),
  });

  if (params.q) search.set('q', params.q);
  if (params.courier) search.set('courier', params.courier);
  if (params.deliveryType) search.set('deliveryType', params.deliveryType);
  if (params.milestone) search.set('milestone', params.milestone);

  return request(`/delivery-history/shipments`, search);
}
```

### 6.2 Mapper `filters` → params

```typescript
export function toListApiParams(
  filters: { query: string; courier: string; deliveryType: string; status: string },
  page: number,
  pageSize: number,
): ShipmentsListParams {
  const trimmedQuery = filters.query.trim();
  const q = trimmedQuery.length >= 3 ? trimmedQuery : undefined;

  return {
    limit: pageSize,
    offset: (page - 1) * pageSize,
    q,
    courier: filters.courier !== 'ALL' ? filters.courier : undefined,
    deliveryType: filters.deliveryType !== 'ALL' ? filters.deliveryType : undefined,
    milestone: filters.status !== 'ALL' ? filters.status : undefined,
  };
}
```

### 6.3 React Query

```typescript
const [filters, setFilters] = useState({
  query: '',
  courier: 'ALL',
  deliveryType: 'ALL',
  status: 'ALL',
});
const [page, setPage] = useState(1);
const [debouncedQuery, setDebouncedQuery] = useState('');

useEffect(() => {
  const timer = setTimeout(() => setDebouncedQuery(filters.query), 300);
  return () => clearTimeout(timer);
}, [filters.query]);

const apiParams = toListApiParams(
  { ...filters, query: debouncedQuery },
  page,
  pageSize,
);

const query = useQuery({
  queryKey: ['shipments', apiParams],
  queryFn: () => fetchShipments(apiParams),
  select: toShipmentListViewData,
  placeholderData: keepPreviousData,
});
```

### 6.4 Page / `ShipmentList`

```typescript
onFiltersChange: (next) => {
  setFilters(next);
  setPage(1);
},
pagination: {
  mode: 'server',
  current: page,
  pageSize,
  total,
  onChange: (nextPage, nextPageSize) => {
    setPage(nextPage);
    setPageSize(nextPageSize);
  },
},
```

### 6.5 `searchFields` del DS

Con paginación server y `q` en API: **no depender** de `searchFields` para filtrar en cliente. Esa prop solo tiene sentido en modo mock o paginación client. Dejarla declarada si el DS la exige, pero la búsqueda real va por `q`.

---

## 7. Tipado API sugerido

Agregar a `ShipmentListApiItem`:

```typescript
currentMilestone: string | null;
```

Usar `currentMilestone` para la columna de estado milestone (label vía catálogo). Los campos `status` / `subStatus` siguen disponibles como diagnóstico operativo legacy.

---

## 8. Ejemplos de request

Listado reciente (sin búsqueda):

```http
GET /api/v1/delivery-history/shipments?limit=20&offset=0
```

Prefix por envío EP:

```http
GET /api/v1/delivery-history/shipments?q=EP0&limit=20&offset=0
```

Prefix + filtro milestone:

```http
GET /api/v1/delivery-history/shipments?q=128&courier=ANDREANI&milestone=LAST_MILE_DISPATCHED&limit=20&offset=0
```

Solo filtros (sin `q`):

```http
GET /api/v1/delivery-history/shipments?courier=ENVIOPACK&deliveryType=HD&limit=20&offset=0
```

---

## 9. Entorno local

| Requisito | Valor |
|-----------|-------|
| Projector | `:9200` |
| Variable FE | `NEXT_PUBLIC_SHIPMENTS_API_BASE_URL=http://localhost:9200/api/v1` |
| Datos reales | `MOCKS_ENABLED=false` en `app_config` del projector (reiniciar servicio) |

Con `MOCKS_ENABLED=true` el projector responde mocks que **sí** filtran por `q`/filtros en memoria, pero no ejercitan la base real. Para validar integración end-to-end con índice PostgreSQL, usar mocks apagados.

---

## 10. Checklist de integración

- [ ] `fetchShipments` acepta `q`, `courier`, `deliveryType`, `milestone`
- [ ] `queryKey` de React Query incluye todos los criterios + paginación
- [ ] `onFiltersChange` actualiza state y resetea página
- [ ] Debounce 300 ms solo en `filters.query`
- [ ] No enviar `q` con menos de 3 caracteres
- [ ] Filtros envían códigos de catálogo, no labels
- [ ] `filters.status` del DS mapea a `milestone` en API
- [ ] `pagination.mode: 'server'` y `total` desde respuesta
- [ ] Tipar y mostrar `currentMilestone` (label desde catálogo)
- [ ] No usar filtrado client-side de `searchFields` como sustituto de `q`

---

## 11. Documentos relacionados

| Documento | Contenido |
|-----------|-----------|
| `detail-api-frontend-contract.md` | Lista + detalle + catálogos |
| `blocks-projector.md` Block 11 | Spec backend search (referencia) |
| `local-testing-sql.md` | Queries SQL de validación manual |
