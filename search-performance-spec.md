# Search performance — spec tentativa (bajo esfuerzo / alto impacto)

**Estado:** TENTATIVE — 2026-06-25  
**Alcance:** `shipments-history-projector` — lectura de `GET /delivery-history/shipments`  
**Referencias:** `rfc2.md` §7.3, `blocks-projector.md` Block 11, benchmarks Artillery (`testing-orchestrator/runs/`)

---

## 1. Objetivo

Reducir latencia p95 de búsqueda y listado filtrado bajo carga (stress Artillery) **sin cambiar el contrato HTTP** ni la semántica de paginación ya cerrada en Block 11.

**Baseline medida (local, projector + Postgres Docker):**

| Seed | Perfil   | p50   | p95    | Notas                                      |
|------|----------|-------|--------|--------------------------------------------|
| 15k  | stress   | 8 ms  | 42 ms  | Referencia sana                            |
| 95k  | realistic| 21 ms | 202 ms | Uso backoffice aceptable                   |
| 95k  | stress   | 169 ms| 6.3 s  | Bimodal: cola + prefijos anchos + `COUNT(*)` |

**Meta tentativa (misma infra, seed ≥ 95k):**

| Perfil   | Métrica | Antes (95k) | Meta Phase 1 |
|----------|---------|-------------|--------------|
| realistic| p95     | 202 ms      | ≤ 150 ms     |
| stress   | p95     | 6.3 s       | ≤ 4 s        |
| stress   | p50     | 169 ms      | ≤ 120 ms     |

Las metas son orientativas; el criterio de aceptación es **mejora relativa ≥ 30% en p95 stress** vs run `2026-06-25T22-27-58-077Z` con el mismo seed y hardware.

---

## 2. Contratos que NO se modifican

Estos invariantes son **obligatorios** para cualquier cambio de esta spec:

| Área | Invariante |
|------|------------|
| HTTP | Un solo endpoint: `GET /delivery-history/shipments` (sin `/search`) |
| Query params | `q`, `courier`, `deliveryType`, `milestone`, `limit` (1–100, default 20), `offset` (≥ 0) |
| Guardrails `q` | Mín 3 / máx 64 chars tras `normalizeSearchToken`; `< 3` → **400** `Invalid search query` |
| Match SQL | Solo `LIKE prefix%` sobre `field_value` normalizado; **no** `ILIKE`, **no** `%prefix%` |
| Respuesta | `{ results: ListShipmentsResultItem[], total: number }` — shape Zod actual sin cambios |
| Semántica `total` | Cardinalidad exacta del conjunto filtrado (`is_listable` + prefix + filtros), no total global |
| Orden | `ORDER BY updated_at DESC` sobre `delivery_history_views` |
| Paginación | `LIMIT` / `OFFSET` sobre el conjunto filtrado (keyset queda fuera de Phase 1) |
| Escritura índice | Solo en tx de proyección (`packageNewMilestoneProjection`); no desde use case de list |
| Mapper boundary | Repositorio delega transformaciones al mapper; sin Prisma types en use cases |

---

## 3. Diagnóstico (causa raíz)

### 3.1 Query actual (con `q`)

`DeliveryHistorySearchRepository.searchListableWithPrefix` ejecuta:

1. `SELECT id` con subquery `DISTINCT view_id` + join + filtros + `ORDER BY` + `LIMIT/OFFSET`
2. `SELECT COUNT(*)` con la **misma** subquery (sin `LIMIT`)
3. `findMany` por ids con `include` de `primaryPackage` y `primaryShipment` + reorden en memoria

**Costo dominante en stress 95k:** paso 2 (count exacto) y paso 1 cuando el prefix matchea muchas filas (`EP000`, `3600029906`).

### 3.2 Índices faltantes

`blocks-projector.md` Block 11 define partial indexes sobre `delivery_history_views` para filtros sin `q`. **No están en la migration inicial.**

### 3.3 Hidratación innecesaria

El listado solo necesita columnas ya presentes en `delivery_history_views` (+ fallbacks de courier). Los JOINs a `primaryPackage` / `primaryShipment` existen por `parcelIdentifier` y fallback de `courier`, no por campos ausentes en la view.

---

## 4. Phase 1 — cambios propuestos (bajo esfuerzo / alto impacto)

Orden de implementación recomendado. Cada ítem es independiente salvo P1.2 que se beneficia de P1.1.

### P1.1 — Partial indexes en `delivery_history_views`

**Esfuerzo:** ~1 migration + verificación en schema Prisma  
**Impacto:** Alto en escenarios sin `q` o con filtros (`milestone`, `courier`, `deliveryType`)  
**Contrato:** Sin cambios

```sql
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

**Archivos:** `schema.prisma`, nueva migration incremental (permitida: no es Block 13 particionado).

**Aceptación:**

- `EXPLAIN` de `GET ...?milestone=DELIVERED&courier=ANDREANI` usa índice parcial en plan local con seed ≥ 95k.
- Stress escenario “filter milestone + courier” mejora p95 ≥ 20% vs baseline.

---

### P1.2 — Una sola query SQL para búsqueda con `q` (eliminar round-trips)

**Esfuerzo:** ~1 archivo (`deliveryHistorySearch.repository.ts`) + mapper method opcional  
**Impacto:** Alto — elimina `COUNT` duplicado y tercer `findMany`  
**Contrato:** Sin cambios en HTTP ni en semántica de `total`

**Query objetivo (ilustrativa):**

```sql
SELECT
    v.id,
    v.display_id,
    v.order_id,
    v.external_order_id,
    v.tracking_number,
    v.courier,
    v.delivery_type,
    v.status,
    v.sub_status,
    v.current_milestone,
    v.etd,
    COUNT(*) OVER() AS total_count
FROM delivery_history_views v
INNER JOIN (
    SELECT DISTINCT view_id
    FROM delivery_history_search_index
    WHERE field_value LIKE $1   -- prefix || '%', ya normalizado en app
) s ON s.view_id = v.id
WHERE v.is_listable = true
  AND ($2::text IS NULL OR v.courier = $2)
  AND ($3::text IS NULL OR v.delivery_type = $3)
  AND ($4::text IS NULL OR v.current_milestone = $4)
ORDER BY v.updated_at DESC
LIMIT $5 OFFSET $6;
```

**Reglas de implementación:**

- `total` = `Number(rows[0]?.total_count ?? 0)`; si `rows` vacío → `total = 0`.
- Mapear filas con método dedicado del mapper, p.ej. `toDomainFromListRow`, **sin** `include` de relaciones.
- `parcel` en respuesta: `display_id` (equivale a `package.external_id` en proyección actual) — mismo valor que hoy vía `primaryPackage.externalId`.
- `courier` en respuesta: `v.courier` (ya denormalizado en view en proyección).
- Mantener transacción única si se prefiere; con una query ya no es obligatoria.

**Aceptación:**

- Tests de integración existentes de list/search siguen pasando.
- Misma `total` y mismos `results` que implementación actual para fixtures conocidos (prefix selectivo y broad).
- Stress p95 global mejora ≥ 30% con seed 95k (mismo run Artillery).

---

### P1.3 — Listado sin `q`: `select` mínimo en Prisma

**Esfuerzo:** Bajo — solo `searchListableWithoutPrefix`  
**Impacto:** Medio en “full list page 100” y deep pagination  
**Contrato:** Sin cambios

Reemplazar `findMany` con `include` por `select` explícito de columnas del list item. Mapear con el mismo `toDomainFromListRow` de P1.2.

**Aceptación:**

- Escenarios stress “full list page 100” y “deep pagination” mejoran p95 ≥ 15% vs baseline.

---

### P1.4 — Verificación de plan con `EXPLAIN` (documentación operativa)

**Esfuerzo:** Bajo — script o sección en README del projector  
**Impacto:** Medio (evita regresiones silenciosas)  
**Contrato:** N/A

Documentar dos queries de referencia con seed 95k:

1. Prefix selectivo (`q` = 12+ chars de EP index 0)
2. Prefix broad (`q` = `EP000`)

Criterio: prefix selectivo debe usar `Index Scan` o `Bitmap Index Scan` sobre `ix_delivery_history_search_index_prefix`.

---

## 5. Phase 1 — fuera de alcance (explícito)

| Ítem | Motivo |
|------|--------|
| `total` aproximado o capado | Rompe semántica Block 11 (`total` exacto) |
| Subir mínimo de `q` a 4–5 chars | Cambio de contrato HTTP/guardrails |
| Keyset / cursor pagination | Cambio de contrato de paginación |
| Denormalizar `view_updated_at` en `search_index` | Migration + escritura en proyección; Phase 2 |
| Particionado / TTL search (Block 13) | Diferido post migration inicial estable |
| Índice por `field_type` | Requiere UX o heurística de tipo; Phase 2 |
| Closure tables | No es bottleneck de search |
| GIN / full-text | Contradice estrategia RFC2 prefix B-tree |

---

## 6. Phase 2 — backlog (mayor esfuerzo, evaluar post Phase 1)

Solo si Phase 1 no alcanza metas con seed 450k:

| ID | Cambio | Impacto | Notas |
|----|--------|---------|-------|
| P2.1 | Columnas denormalizadas en `search_index` (`view_updated_at`, `is_listable`, filtros) + índice `(field_value pattern_ops, view_updated_at DESC)` | Muy alto en broad prefix | Migration + mapper de proyección |
| P2.2 | Indexar EP + last-mile como filas `TRACKING` distintas cuando coexisten | Medio | Caso marketplace RFC2 §2.1 |
| P2.3 | Keyset pagination (`updated_at`, `id`) | Alto en offset grande | Requiere acuerdo FE |
| P2.4 | Block 13: particionado + TTL de `search_index` | Alto a 450k+ | Post estabilización modelo |

---

## 7. Validación

### 7.1 Automatizada (existente)

```bash
# Pre-requisitos: seed ≥ 95k, MOCKS_ENABLED=false, projector en :9200
cd testing-orchestrator
npm run search-load
```

Comparar `artillery/summary.json` entre runs antes/después en la **misma máquina** y mismo `seedCount`.

### 7.2 Checks manuales mínimos

| Caso | Request | Esperado |
|------|---------|----------|
| Prefix selectivo | `?q=<epSelectivePrefix>&limit=20` | `total >= 1`, p95 < 500 ms local |
| Prefix broad | `?q=EP000&limit=100` | `total` = seed count, sin 5xx |
| Filtros | `?milestone=DELIVERED&courier=ANDREANI` | `total` coherente, usa partial index |
| Guardrail | `?q=EP` | **400** |
| Sin q | `?limit=20&offset=0` | Orden por `updated_at` desc |

### 7.3 Regresión de contrato

- OpenAPI / Zod `ListShipmentsOutputSchema` sin cambios de campo.
- `testing-orchestrator` processors `validateSearchResponse` / `validateListResponse` sin cambios.

---

## 8. Riesgos y mitigaciones

| Riesgo | Mitigación |
|--------|------------|
| `COUNT(*) OVER()` en broad prefix sigue siendo O(n) | Phase 1 aún elimina un pass completo; P2.1 si no alcanza |
| `parcel` ≠ `display_id` en datos reales futuros | Mantener fallback a `primaryPackage.externalId` en mapper si `display_id` null |
| Migration de índices en tabla grande (450k) | `CREATE INDEX CONCURRENTLY` en prod; migration dev puede ser blocking |
| Bimodalidad por pool Postgres saturado | Documentar `max_connections` / pool Prisma; no confundir con query lento |

---

## 9. Checklist de implementación

- [ ] **P1.1** Migration partial indexes + `schema.prisma`
- [ ] **P1.2** Refactor `searchListableWithPrefix` → query única + mapper list row
- [ ] **P1.3** Refactor `searchListableWithoutPrefix` → `select` mínimo
- [ ] **P1.4** Documentar `EXPLAIN` de referencia
- [ ] Re-ejecutar `npm run search-load` con seed 95k y archivar summary
- [ ] (Opcional) Re-ejecutar con seed 250k/450k cuando esté disponible

---

## 10. Resumen ejecutivo

**Hacer ahora (Phase 1):** índices parciales ya especificados en Block 11 + colapsar la búsqueda a **una query** con `COUNT(*) OVER()` + dejar de hacer JOINs en el path de listado.

**No tocar:** contrato HTTP, guardrails de `q`, semántica de `total` exacto, estrategia prefix B-tree del RFC2.

**Medir:** Artillery realistic + stress, mismo seed, misma máquina — comparar p50/p95 por endpoint en `stress.json`.
