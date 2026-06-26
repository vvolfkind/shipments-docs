# Validaciones SQL — E2E local

Queries de diagnóstico para el flujo de `local-testing-guide.md`. Reemplazá `JOURNEY_UUID`, `EP#########N-01`, etc.

---

## 1. Config estática

```sql
```sql
SELECT key, value FROM app_config;
-- Projector mocks: MOCKS_ENABLED (boolean) — list, detail, milestones catalog
```
WHERE key IN ('JOURNEY_RECONCILIATION_CHUNK_SIZE', 'INBOX_OUTBOX_MAX_ATTEMPTS');

SELECT csm.event, csm.sub_status, COUNT(csmk.id) AS key_count
FROM carrier_status_mappings csm
JOIN carrier_status_mapping_keys csmk ON csmk.mapping_id = csm.id
WHERE csm.source = 'Andreani' AND csm.is_active = true
GROUP BY csm.event, csm.sub_status
ORDER BY csm.event, csm.sub_status;
```

---

## 2. EP ready-to-collect

```sql
SELECT COUNT(*) AS ep_ready_count
FROM packages p
JOIN package_events pe ON pe.package_id = p.id
WHERE p.source_system = 'ENVIOPACK'
  AND pe.topic = 'ep-ready-to-collect-orders'
  AND pe.event_name = 'EP_READY';

SELECT p.external_id, p.current_status
FROM packages p
JOIN package_events pe ON pe.package_id = p.id
WHERE pe.event_name = 'EP_READY'
ORDER BY pe.occurred_at DESC
LIMIT 5;

SELECT p.external_id, sh.trail_type, sh.carrier_code, ps.exited_at
FROM packages p
JOIN package_shipments ps ON ps.package_id = p.id
JOIN shipments sh ON sh.id = ps.shipment_id
WHERE p.external_id ~ '^EP\d{9}N$'
LIMIT 5;
```

---

## 3. Journeys

```sql
SELECT id, external_id, status, total_receipt_items, processed_receipt_items
FROM journeys
WHERE source_system = 'WMI'
ORDER BY announced_at;

SELECT journey_id, status, COUNT(*)
FROM journey_receipt_items
GROUP BY journey_id, status;

SELECT j.external_id AS entry_order,
       jri.package_external_id,
       jri.last_mile_id
FROM journey_receipt_items jri
JOIN journeys j ON j.id = jri.journey_id
ORDER BY j.external_id, jri.line_index
LIMIT 10;
```

---

## 4. Reconciliación

```sql
SELECT id, status, processed_receipt_items, total_receipt_items
FROM journeys
WHERE id = 'JOURNEY_UUID';

SELECT status, COUNT(*) FROM journey_receipt_items
WHERE journey_id = 'JOURNEY_UUID' GROUP BY status;

SELECT COUNT(*) FROM journey_packages WHERE journey_id = 'JOURNEY_UUID';

SELECT p.external_id, pil.identifier_type, pil.identifier
FROM journey_packages jp
JOIN packages p ON p.id = jp.package_id
LEFT JOIN package_identifier_links pil ON pil.package_id = p.id
WHERE jp.journey_id = 'JOURNEY_UUID'
ORDER BY p.external_id, pil.identifier_type
LIMIT 20;

SELECT pm.journey_announced_at, pm.oe_launch_at
FROM package_milestones pm
JOIN journey_packages jp ON jp.package_id = pm.package_id
WHERE jp.journey_id = 'JOURNEY_UUID'
LIMIT 5;
```

---

## 5. OE_LAUNCH

```sql
SELECT event_type, status FROM inbox_events
WHERE event_type = 'WM2_RS_MAIN_OE_LAUNCH'
ORDER BY created_at DESC LIMIT 3;

SELECT p.external_id, pm.oe_launch_at
FROM journey_packages jp
JOIN packages p ON p.id = jp.package_id
JOIN package_milestones pm ON pm.package_id = p.id
WHERE jp.journey_id = 'JOURNEY_UUID'
LIMIT 5;

SELECT p.external_id, sh.trail_type, sh.tracking_number, ps.exited_at
FROM packages p
JOIN package_shipments ps ON ps.package_id = p.id
JOIN shipments sh ON sh.id = ps.shipment_id
WHERE p.id IN (SELECT package_id FROM journey_packages WHERE journey_id = 'JOURNEY_UUID')
ORDER BY p.external_id, sh.trail_type
LIMIT 10;

SELECT status, COUNT(*) FROM outbox_ingestor
WHERE queue_name = 'shipments-history-api-to-projector'
GROUP BY status;

SELECT label_identifier, status, LEFT(payload, 120)
FROM outbox_ingestor
WHERE payload LIKE '%WAREHOUSE_RECEIVED%'
ORDER BY created_at DESC LIMIT 3;
```

### Projector (post OE_LAUNCH)

```sql
SELECT event_type, status, COUNT(*)
FROM inbox_events
WHERE event_type = 'PACKAGE_NEW_MILESTONE' AND source = 'SHIPMENTS_HISTORY_API'
GROUP BY 1, 2;

SELECT COUNT(*) FROM delivery_history_views WHERE is_listable = true;

SELECT display_id, status, current_milestone, courier, tracking_number,
       metadata->>'emissionTrigger' AS trigger
FROM delivery_history_views
WHERE is_listable = true
ORDER BY last_occurred_at DESC
LIMIT 5;
```

---

## 6. Andreani OOLL

```sql
SELECT status, COUNT(*) FROM inbox_events
WHERE event_type = 'ANDREANI_EVENTS'
  AND created_at > NOW() - INTERVAL '30 minutes'
GROUP BY status;

SELECT p.external_id, pm.picked_up_by_carrier_at, pm.in_transit_at, pm.delivered_at
FROM packages p
JOIN package_milestones pm ON pm.package_id = p.id
WHERE p.external_id = 'EP#########N-01';

SELECT pe.sub_status, pe.occurred_at
FROM package_events pe
JOIN packages p ON p.id = pe.package_id
WHERE p.external_id = 'EP#########N-01' AND pe.topic = 'andreani-events'
ORDER BY pe.occurred_at;
```

### Projector

```sql
SELECT display_id, status, current_milestone, last_occurred_at
FROM delivery_history_views
WHERE display_id = 'EP#########N-01';

SELECT pe.sub_status, pe.canonical_status, pe.occurred_at
FROM package_events pe
JOIN packages p ON p.id = pe.package_id
WHERE p.external_id = 'EP#########N-01'
ORDER BY pe.occurred_at;
```
