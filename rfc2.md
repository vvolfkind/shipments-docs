1. Objetivo y Alcance
Definir la arquitectura de datos y la estrategia de persistencia para un sistema de trazabilidad de alta disponibilidad, centrado en la normalización y centralización de eventos logísticos. El alcance de este documento se limita a la infraestructura de datos necesaria para resolver la fragmentación en órdenes multi-bulto, cubriendo desde la ingesta masiva y el procesamiento asíncrono, hasta la creación de proyecciones normalizadas para análisis dinámico y búsqueda.

El foco está en asegurar el Data Lineage (vínculo Orden <-> Bulto <-> Tracking) mediante el procesamiento de flujos de datos desordenados, garantizando que la información esté disponible y normalizada para herramientas de búsqueda (Search) y visibilidad granular, sin comprometer la performance de ingesta.

2. Contexto de Negocio
2.1. Estado actual
Frávega gestiona entregas a nivel orden (Delivery ID): un único tracking y un único estado por orden. No existe el concepto operativo de envío ni de bulto como unidades trazables independientes dentro de la misma orden.

A nivel de modelo de datos, TMS/Courier Manager mantiene un único envío en curso por Delivery ID. Cuando llega un segundo envío asociado a la misma orden -por faltante, por reenvío, o porque el operador logístico opera con paquetería y genera un tracking por bulto- el sistema pisa el envío anterior o pierde el dato por no poder relacionarlo a ninguna orden. Todas las novedades posteriores de los envíos no mapeados se descartan y se pierde la trazabilidad.

El mismo problema ocurre con el tracking de Envíopack (tracking EP del primer tramo seller → CD): al asignarse el operador logístico final, el tracking del OL reemplaza al de EP en lugar de coexistir con él.

La eficiencia de entregar múltiples productos al mismo cliente en un solo recorrido se logra hoy principalmente con flete propio, por falta de trazabilidad multi bulto con operadores logísticos externos.

Delivery Manager admite un único tracking/link por orden.

2.1. Modelos de operadores logísticos
Los OOLL externos se dividen en dos modelos según su contrato:

Modelo guía única: aplica por ejemplo a productos de gran morfología (MAXI, BIG, BIGGER) que se despachan en múltiples bultos físicos por su naturaleza (ej. un sommier = colchón + sommier + paquete con patas). Un tracking por envío, puede contener N bultos. Las etiquetas físicas se numeran secuencialmente (ej. XXXXXX-001, XXXXXX-002), pero el seguimiento se hace a nivel envío consolidado. El OL entrega o no entrega el envío completo; no gestiona entregas parciales entre bultos de un mismo envío.

Modelo paquetería (ej. Andreani contrato paquetería, Moova): un tracking independiente por bulto, estados independientes por bulto. Puede habilitar entrega parcial: una orden con tres bultos puede tener dos entregados y uno en tránsito simultáneamente. Frávega busca derivar el estado de la orden a partir de los estados individuales de cada bulto.

En ambos modelos, la unidad de seguimiento del OL es el envío desde su perspectiva; la diferencia es si ese envío puede contener múltiples bultos trazables o no. El concepto de “multi bulto” para Frávega significa que una orden puede resolverse en N envíos, independientemente de cómo cada OOLL modele internamente sus bultos. Los contratos actuales con los OOLL no garantizan soporte de entrega parcial en todos los casos, pero se debe apuntar/asumir que todos van a evolucionar hacia ese modelo preparando el sistema para soportar entrega parcial como caso de primer nivel, no como caso border.

2.4. Problema a resolver
El sistema debe:

Soportar N envíos por Delivery ID, cada uno con su propia timeline de tracking.

Reconstruir el historial completo de un envío a partir de eventos que pueden llegar fuera de orden o con retraso.

Derivar el estado de la orden a partir del estado agregado de sus envíos, incluyendo estados parciales (ej. “un bulto entregado, uno en tránsito”).

Exponer esta visibilidad renderizando la timeline de los tracking individuales

Para marketplace (EnvioPack), se debe persistir el tracking EP y el tracking del OL sin que uno pise al otro. El flete propio y Retail quedan fuera del primer MVP pero la solución debe ser estructuralmente análoga para absorberlos en iteraciones posteriores.

3. Arquitectura de Servicios
El sistema se divide en dos servicios con responsabilidades físicas y lógicas no superpuestas.

Servicio 1 — Ingesta (Write-Heavy)
Escucha Fenix Queues y potencialmente Kafka/RabbitMQ directo. Valida idempotencia y persiste el Event Store. No aplica lógica de negocio. Diseñado para no bloquearse nunca.

Servicio 2 — Delivery History (Read-Heavy + Logic)
Contiene las proyecciones CQRS, resuelve jerarquías de bultos, expone la API y el buscador para Delivery Manager. Es donde reside la lógica de negocio de logística.


5. Modelo de Entidades
Jerarquía de cinco niveles para resolver el conflicto entre abstracción logística y comercial:

Order — Nodo comercial, vínculo con el cliente.

Delivery — Unidad raíz de compromiso de entrega. Puede existir sin Order asociada. Un delivery → N shipments

Shipment — Contrato con el operador logístico (ej. Andreani, Enviopack).

Package — Identidad física del bulto con sus identificadores (tracking_id, códigos de operador).

Item — SKU y propiedades, si correspondiera.

Todas las entidades manejan enums de STATUS y SUB-STATUS.

5.1. Lógica de Reconciliación (Order–Delivery)
El Delivery se crea en el Read Model de forma autónoma al recibir el primer evento de tracking. El campo order_id es una referencia lógica, no un prerequisito de existencia. Si el evento llega antes que la orden sea reportada por el ERP/OMS, el Delivery queda en estado ORDER_PENDING. Cuando el evento comercial impacta, el sistema vincula retroactivamente todos los Deliveries asociados mediante order_id.

6. Modelo de Escritura — Ingestor
6.1. Patrón Inbox/Outbox
El Ingestor implementa tres etapas dentro de la misma transacción:

Inbox (Ingesta Atómica): Persistencia inmediata del evento entrante. Sin lógica de negocio. Latencia mínima garantizada.

Worker de Procesamiento (Polling con Batching): Loop asíncrono que consume el Inbox procesando lotes (ej. 500 registros) de forma atómica.

Outbox (Transactional): En la misma transacción de procesamiento, el evento normalizado se persiste en outbox_events. Un relay asíncrono lo emite hacia Fenix Queues para consumo del Delivery Engine.

6.2. Escenarios de Inbox
La estructura del inbox diverge según el origen de los eventos. El Read Model y el Outbox son idénticos en ambos casos.

Escenario A — Fuentes Internas Estandarizadas

El servicio upstream ya validó y serializó el payload en un contrato conocido. El inbox tiene columnas tipadas; el worker ejecuta una proyección directa al Read Model.



CREATE TYPE event_status AS ENUM (
    'PENDING', 'PROCESSING', 'PROCESSED', 'FAILED'
);
-- A revisar segun convencion actual de DM
CREATE TYPE shipment_status AS ENUM (
    'CREATED', 'LABEL_GENERATED', 'PICKED_UP', 'IN_TRANSIT',
    'OUT_FOR_DELIVERY', 'DELIVERED', 'RETURNED', 'CANCELLED', 'EXCEPTION'
);
CREATE TABLE inbox_events (
    id                UUID         NOT NULL DEFAULT gen_random_uuid(),
    event_type        VARCHAR(64)  NOT NULL,
    -- Payload pre-validado por servicio upstream
    order_id          VARCHAR(64)  NOT NULL,
    carrier_code      VARCHAR(32)  NOT NULL,
    tracking_number   VARCHAR(128) NOT NULL,
    shipment_status   VARCHAR(64),
    status_detail     TEXT,
    occurred_at       TIMESTAMPTZ,          -- Event Time del operador
    shipment_payload  JSONB        NOT NULL DEFAULT '{}',
    -- Anchor de idempotencia
    external_event_id VARCHAR(256) NOT NULL,
    -- Worker claim
    status            event_status NOT NULL DEFAULT 'PENDING',
    version           INT          NOT NULL DEFAULT 0,
    locked_at         TIMESTAMPTZ,
    locked_by         VARCHAR(253),         -- FQDN K8s pod
    -- Retry
    retry_count       INT          NOT NULL DEFAULT 0,
    max_retries       INT          NOT NULL DEFAULT 5,
    last_error        TEXT,
    created_at        TIMESTAMPTZ  NOT NULL DEFAULT now(),
    updated_at        TIMESTAMPTZ  NOT NULL DEFAULT now(),
    processed_at      TIMESTAMPTZ,
    CONSTRAINT pk_inbox_events PRIMARY KEY (id, created_at),
    CONSTRAINT uq_inbox_events_idempotency
        UNIQUE (carrier_code, external_event_id, created_at)
) PARTITION BY RANGE (created_at);
-- Partial index: solo filas accionables. Se encoge de manera automatica a medida que se procesan eventos.
CREATE INDEX ix_inbox_events_poll
    ON inbox_events (created_at)
    WHERE status IN ('PENDING', 'FAILED');
-- Reprocessing por orden
CREATE INDEX ix_inbox_events_order
    ON inbox_events (order_id, created_at DESC);
Escenario B — Raw Provider Passthrough

Para operadores logísticos que envían webhooks con schemas propios. El inbox es un pipe en el que la extracción y normalización ocurre en el worker.



CREATE TABLE inbox_events (
    id                UUID         NOT NULL DEFAULT gen_random_uuid(),
    carrier_code      VARCHAR(32)  NOT NULL,
    webhook_event     VARCHAR(64),
    -- Payload sin tocar, exactamente como llegó
    raw_payload       JSONB        NOT NULL,
    http_headers      JSONB,
    received_url      VARCHAR(512),
    external_event_id VARCHAR(256) NOT NULL,
    status            event_status NOT NULL DEFAULT 'PENDING',
    version           INT          NOT NULL DEFAULT 0,
    locked_at         TIMESTAMPTZ,
    locked_by         VARCHAR(253),
    retry_count       INT          NOT NULL DEFAULT 0,
    max_retries       INT          NOT NULL DEFAULT 5,
    last_error        TEXT,
    created_at        TIMESTAMPTZ  NOT NULL DEFAULT now(),
    updated_at        TIMESTAMPTZ  NOT NULL DEFAULT now(),
    processed_at      TIMESTAMPTZ,
    CONSTRAINT pk_inbox_events PRIMARY KEY (id, created_at),
    CONSTRAINT uq_inbox_events_idempotency
        UNIQUE (carrier_code, external_event_id, created_at)
) PARTITION BY RANGE (created_at);
CREATE INDEX ix_inbox_events_poll
    ON inbox_events (created_at)
    WHERE status IN ('PENDING', 'FAILED');
-- Debugging: "todos los eventos raw de Andreani hoy"
CREATE INDEX ix_inbox_events_carrier
    ON inbox_events (carrier_code, created_at DESC);
Tablas adicionales exclusivas del Escenario B:



-- Tabla de traducción: raw status del operador → canonical shipment_status
CREATE TABLE carrier_status_mappings (
    id               UUID            NOT NULL DEFAULT gen_random_uuid(),
    carrier_code     VARCHAR(32)     NOT NULL,
    raw_status       VARCHAR(128)    NOT NULL,
    canonical_status shipment_status NOT NULL,
    is_terminal      BOOLEAN         NOT NULL DEFAULT FALSE,
    CONSTRAINT pk_carrier_status_mappings PRIMARY KEY (id),
    CONSTRAINT uq_carrier_status_mappings UNIQUE (carrier_code, raw_status)
);
-- Audit trail de la extracción JSONB → Read Model
CREATE TABLE extraction_logs (
    id                    UUID        NOT NULL DEFAULT gen_random_uuid(),
    inbound_event_id      UUID        NOT NULL,
    carrier_code          VARCHAR(32) NOT NULL,
    extracted_order_id    VARCHAR(64),
    extracted_tracking    VARCHAR(128),
    extraction_status     VARCHAR(32) NOT NULL,  -- 'SUCCESS', 'PARTIAL', 'FAILED'
    extraction_errors     JSONB,
    jsonpath_version      VARCHAR(16),           -- versión de la lógica de extracción
    created_at            TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT pk_extraction_logs PRIMARY KEY (id)
);
CREATE INDEX ix_extraction_logs_event
    ON extraction_logs USING btree (inbound_event_id);
Dimensión

Escenario A: Estandarizado

Escenario B: Raw Passthrough

Schema del inbox

Columnas tipadas

JSONB + http_headers

Clave de idempotencia

carrier_code + external_event_id

carrier_code + external_event_id

Complejidad del worker

Baja — validar y proyectar

Alta — extraer, mapear status, manejar schema drift

Tablas extra

Ninguna

carrier_status_mappings, extraction_logs

Mejor para

Microservicios internos controlados

Operadores externos con schemas impredecibles

El escenario A es el pre-seleccionado, dado que ya existen los adapters de TMS.

El Escenario B se menciona dado que gana desacoplamiento total de los schemas de los OOLL. Cuando un carrier cambia su formato, solo se actualiza la lógica de extracción y el inbox nunca rompe. Sin embargo es mas caro en recursos (CPU en el worker)

6.3. Outbox
Idéntico en ambos escenarios:



CREATE TABLE outbox_events (
    id             UUID         NOT NULL DEFAULT gen_random_uuid(),
    aggregate_type VARCHAR(64)  NOT NULL,
    aggregate_id   UUID         NOT NULL,
    event_type     VARCHAR(64)  NOT NULL,
    payload        JSONB        NOT NULL,
    status         event_status NOT NULL DEFAULT 'PENDING',
    version        INT          NOT NULL DEFAULT 0,
    locked_at      TIMESTAMPTZ,
    locked_by      VARCHAR(253),
    retry_count    INT          NOT NULL DEFAULT 0,
    max_retries    INT          NOT NULL DEFAULT 5,
    last_error     TEXT,
    created_at     TIMESTAMPTZ  NOT NULL DEFAULT now(),
    updated_at     TIMESTAMPTZ  NOT NULL DEFAULT now(),
    processed_at   TIMESTAMPTZ,
    CONSTRAINT pk_outbox_events PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);
CREATE INDEX ix_outbox_events_poll
    ON outbox_events (created_at)
    WHERE status IN ('PENDING', 'FAILED');
6.4. Protocolo de Claim del Worker
Decisión abierta: existen dos mecanismos viables para el claim concurrente. Deben evaluarse antes de ser implementados.

Opción A — Optimistic Locking con version



-- Paso 1: Poll sin lock (el partial index cubre exactamente este query)
SELECT id, version, carrier_code, external_event_id
FROM inbox_events
WHERE status = 'PENDING'
  AND created_at >= now() - INTERVAL '24 hours'
ORDER BY created_at ASC
LIMIT 50;
-- Paso 2: Claim atómico — el WHERE version = $expected es la gate de contención.
-- Si otro pod ganó primero: version ya cambió → affected_rows = 0. Sin excepción, sin deadlock.
UPDATE inbox_events
SET status     = 'PROCESSING',
    version    = version + 1,
    locked_at  = now(),
    locked_by  = $pod_name,
    updated_at = now()
WHERE id         = $event_id
  AND created_at = $created_at
  AND version    = $expected_version
  AND status     = 'PENDING';
-- Si affected_rows = 0: otro pod ganó. Saltar, no reintentar.
Opción B — FOR UPDATE SKIP LOCKED



SELECT id, carrier_code, external_event_id
FROM inbox_events
WHERE status = 'PENDING'
  AND created_at >= now() - INTERVAL '24 hours'
ORDER BY created_at ASC
LIMIT 50
FOR UPDATE SKIP LOCKED;
-- Cada instancia bloquea su batch de forma exclusiva y transparente para el resto.
 

Optimistic Lock (version)

FOR UPDATE SKIP LOCKED

Contención

Resuelto en UPDATE; sin locks en poll

Lock pesimista por fila durante la transacción

Complejidad del worker

Chequear affected_rows por evento

Simple: si el worker lo obtiene, es suyo

Escalabilidad

Mejor en escenarios de alta concurrencia de lectura

Mejor en escenarios de baja latencia de claim

Ventaja adicional

version provee audit de posibles mutations

Menos código en el worker

6.5. Ciclo Completo de Procesamiento


BEGIN;
-- 1. Upsert en el Read Model
INSERT INTO shipments (order_id, carrier_code, tracking_number, current_status, ...)
VALUES ($order_id, $carrier_code, $tracking, $status, ...)
ON CONFLICT (carrier_code, tracking_number)
DO UPDATE SET
    current_status = EXCLUDED.current_status,
    updated_at     = now();
-- 2. Insertar checkpoint de tracking
INSERT INTO tracking_details (shipment_id, status_code, occurred_at, ...)
VALUES ($shipment_id, $status_code, $occurred_at, ...)
ON CONFLICT (shipment_id, status_code, occurred_at) DO NOTHING;
-- 3. Audit log
INSERT INTO status_logs (shipment_id, previous_status, new_status, changed_by, inbound_event_id)
VALUES ($shipment_id, $prev, $new, $pod_name, $event_id);
-- 4. Outbox en la misma transacción
INSERT INTO outbox_events (aggregate_type, aggregate_id, event_type, payload)
VALUES ('shipment', $shipment_id, 'shipment.status_changed', $event_json);
-- 5. Marcar inbox como PROCESSED
UPDATE inbox_events
SET status       = 'PROCESSED',
    version      = version + 1,
    processed_at = now(),
    updated_at   = now()
WHERE id         = $event_id
  AND created_at = $created_at;
COMMIT;
Path de falla:



UPDATE inbox_events
SET status      = CASE
                    WHEN retry_count + 1 >= max_retries THEN 'FAILED'
                    ELSE 'PENDING'
                  END,
    version     = version + 1,
    retry_count = retry_count + 1,
    last_error  = $error_message,
    locked_at   = NULL,
    locked_by   = NULL,
    updated_at  = now()
WHERE id         = $event_id
  AND created_at = $created_at;
Recovery de stale locks — cron job (Fenix) que libera eventos colgados en PROCESSING por crash de pod:



UPDATE inbox_events
SET status     = 'PENDING',
    version    = version + 1,
    locked_at  = NULL,
    locked_by  = NULL,
    updated_at = now()
WHERE status    = 'PROCESSING'
  AND locked_at < now() - INTERVAL '5 minutes';
6.6. Event Time vs. Processing Time
La timeline de un bulto se ordena por occurred_at (Event Time: momento en que el evento ocurrió en el operador), no por created_at (Processing Time: momento en que llegó al sistema). Esto implica un mapeo explícito por operador: identificar y documentar qué campo del payload de cada carrier representa el timestamp del evento real. La diferencia entre ambos es la Lateness. Se puede definir un Watermark como umbral temporal para decidir cuándo cerrar una ventana de consolidación.

7. Modelo de Lectura — Delivery History
7.1. Read Model Schema


-- Shipments: join point canónico entre órdenes y bultos
CREATE TABLE shipments (
    id                 UUID            NOT NULL DEFAULT gen_random_uuid(),
    order_id           VARCHAR(64)     NOT NULL,
    carrier_code       VARCHAR(32)     NOT NULL,
    tracking_number    VARCHAR(128)    NOT NULL,
    current_status     shipment_status NOT NULL DEFAULT 'CREATED',
    shipped_at         TIMESTAMPTZ,
    delivered_at       TIMESTAMPTZ,
    estimated_delivery TIMESTAMPTZ,
    recipient_name     VARCHAR(256),
    shipping_address   JSONB,
    weight_grams       INT,
    metadata           JSONB           NOT NULL DEFAULT '{}',
    created_at         TIMESTAMPTZ     NOT NULL DEFAULT now(),
    updated_at         TIMESTAMPTZ     NOT NULL DEFAULT now(),
    CONSTRAINT pk_shipments PRIMARY KEY (id),
    CONSTRAINT uq_shipments_carrier_tracking UNIQUE (carrier_code, tracking_number)
);
-- "Todos los shipments de la orden X" — B-Tree, O(log n), soporta ORDER BY
CREATE INDEX ix_shipments_order_id
    ON shipments USING btree (order_id);
-- "Buscar shipment por tracking number"
CREATE INDEX ix_shipments_tracking_number
    ON shipments USING btree (tracking_number);
-- Dashboard: filtrar bultos activos. Partial index excluye ~80% de filas terminales.
CREATE INDEX ix_shipments_status
    ON shipments USING btree (current_status)
    WHERE current_status NOT IN ('DELIVERED', 'CANCELLED');
-- Tracking Details: cada checkpoint del carrier
CREATE TABLE tracking_details (
    id                 UUID        NOT NULL DEFAULT gen_random_uuid(),
    shipment_id        UUID        NOT NULL,
    status_code        VARCHAR(64) NOT NULL,
    status_description TEXT,
    location           VARCHAR(256),
    occurred_at        TIMESTAMPTZ NOT NULL,  -- Event Time
    raw_carrier_status VARCHAR(128),
    created_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT pk_tracking_details PRIMARY KEY (id),
    CONSTRAINT fk_tracking_details_shipment
        FOREIGN KEY (shipment_id) REFERENCES shipments (id),
    CONSTRAINT uq_tracking_details_dedup
        UNIQUE (shipment_id, status_code, occurred_at)
);
-- "Timeline de tracking para shipment X, más reciente primero" — index-only scan
CREATE INDEX ix_tracking_details_shipment
    ON tracking_details USING btree (shipment_id, occurred_at DESC);
-- Status Logs: audit trail de mutaciones del Read Model
CREATE TABLE status_logs (
    id               UUID            NOT NULL DEFAULT gen_random_uuid(),
    shipment_id      UUID            NOT NULL,
    previous_status  shipment_status,
    new_status       shipment_status NOT NULL,
    changed_at       TIMESTAMPTZ     NOT NULL DEFAULT now(),
    changed_by       VARCHAR(253)    NOT NULL,
    reason           TEXT,
    inbound_event_id UUID,
    CONSTRAINT pk_status_logs PRIMARY KEY (id),
    CONSTRAINT fk_status_logs_shipment
        FOREIGN KEY (shipment_id) REFERENCES shipments (id)
);
CREATE INDEX ix_status_logs_shipment
    ON status_logs USING btree (shipment_id, changed_at DESC);
7.2. Justificación de Índices
Read Model — B-Tree en todos los casos:

Índice

Justificación

ix_shipments_order_id

Queries de Post-Venta siempre son WHERE order_id = ?. B-Tree da O(log n) y soporta ORDER BY. Hash serviría para igualdad pura pero no para range predicates si los requisitos evolucionan.

ix_shipments_tracking_number

Lookups de integración con carriers: WHERE tracking_number = ?. B-Tree también soporta prefix scans si los tracking numbers comparten prefijos de carrier.

ix_shipments_status (partial)

Filtra bultos activos en el dashboard. Partial WHERE status NOT IN ('DELIVERED','CANCELLED') mantiene el árbol pequeño: ~80% de filas son terminales y quedan excluidas.

ix_tracking_details_shipment

Compuesto (shipment_id, occurred_at DESC) — sirve el query “timeline de shipment X, más reciente primero” como index-only scan.

Inbox/Outbox — Indexing mínimo:

El inbox es una cola transitoria, no una superficie de consulta. Sobre-indexarlo genera dos problemas durante picos 8x (eventos T1): 

Cada INSERT actualiza cada índice. Un GIN sobre JSONB sería catastrófico. Bloat bajo alta rotación: filas pasan PENDING → PROCESSING → PROCESSED rápido. 

Cada UPDATE crea dead tuples en cada índice. 

El único índice parcial WHERE status IN ('PENDING','FAILED') es el punto de equilibrio: pequeño (solo filas accionables), sirve al único query pattern que ejecutan los workers, y se encoge automáticamente a medida que se procesan eventos.

7.3. Buscador Full-Text
Para búsqueda de texto se implementa una tabla de búsqueda:

Tabla de Búsqueda: shipment_search_index.

Índice Optimizado: B-tree con varchar_pattern_ops para soportar búsquedas por prefijo ultra rápidas.

Mantenimiento Asíncrono: La actualización del índice ocurre durante la fase de refinamiento del bulto (Service B), asegurando que la búsqueda no bloquee la ingesta de webhooks.



CREATE TABLE shipment_search_index (
    field_value   VARCHAR(256) NOT NULL, -- DNI, Tracking, OrderID, etc.
    shipment_id   UUID NOT NULL,
    field_type    VARCHAR(16) NOT NULL,
    updated_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (field_value, shipment_id) -- Ordenado para búsquedas por prefijo
);
CREATE INDEX ix_search_index_prefix ON shipment_search_index(field_value varchar_pattern_ops);
7.4. Order Splitting y Jerarquías de Bultos
Para resolver la fragmentación de órdenes en múltiples unidades trazables, se adopta un modelo de Closure Tables como estructura de datos primaria para el linaje. Mientras que las Recursive CTEs (Common Table Expressions) calculan la jerarquía en tiempo de ejecución (CPU-bound), las Closure Tables pre-calculan y almacenan todas las relaciones entre ancestros y descendientes en una tabla auxiliar.

Performance: Esta estrategia permite consultar el linaje completo de un bulto o recuperar todos los bultos asociados a una orden comercial en $O(1)$, eliminando la variabilidad de performance que presentan las queries recursivas ante jerarquías profundas o desbalanceadas.

Implementación: Se mantendrá una tabla de búsqueda que desacople el shipment_id de sus distintos niveles de asociación.

Trade-off: Se acepta un mayor consumo de almacenamiento en disco a cambio de una latencia de lectura constante y predecible, soportando alta concurrencia en operaciones de búsqueda.

En el MVP, la jerarquía con JOINs directos por FK: Order → Delivery → Shipment → Package → Item hace que no sean necesarias Closure Tables. Esto funciona porque la profundidad es fija (5 niveles) y conocida. Las Closure Tables serían necesarias si:

Order splitting dinámico: Una orden se fragmenta en sub-órdenes que a su vez se fragmentan (profundidad variable)

Consultas de linaje arbitrario: "¿De qué orden original viene este bulto que pasó por 3 splits?"

Volumen alto de queries de ancestros/descendientes: Donde los JOINs encadenados degradan performance.

8. Persistencia: Particionado y Ciclo de Vida
8.1. Estrategia de Particionado
inbox_events y outbox_events se particionan por RANGE (created_at), mensualmente.

Preocupación

Cómo lo resuelve el particionado

Bloat por picos 8x

Los rows PROCESSED acumulan solo en la partición del mes actual. Las particiones viejas se congelan (“frozen”, sin writes = sin bloat).

Presión de VACUUM

Autovacuum trabaja sobre tablas pequeñas por partición en vez de un heap masivo. Los dead tuples del ciclo de claim se limpian más rápido.

Costo de cleanup

Detach/drop de una partición es O(1) metadata. Borrar millones de rows de una tabla monolítica bloquea y genera bloat adicional.

Partition pruning

El WHERE created_at >= now() - INTERVAL '24 hours' en el poll omite todas las particiones excepto la actual (y quizás la anterior). Los workers nunca escanean meses archivados.

Constraint de idempotencia

La unique constraint incluye created_at (partition key) como requiere PostgreSQL. Dentro de una ventana temporal razonable, los duplicados se detectan. Si un carrier repite un evento meses después, aterriza en una nueva partición; la unique constraint del Read Model (carrier_code, tracking_number) actúa como segunda línea de defensa.

8.2. Creación Automática de Particiones


CREATE OR REPLACE PROCEDURE create_monthly_partitions(
    p_table_name TEXT,
    p_months_ahead INT DEFAULT 2
)
LANGUAGE plpgsql AS $$
DECLARE
    v_start DATE;
    v_end   DATE;
    v_name  TEXT;
BEGIN
    FOR i IN 0..p_months_ahead LOOP
        v_start := date_trunc('month', now()) + (i || ' months')::INTERVAL;
        v_end   := v_start + INTERVAL '1 month';
        v_name  := format('%s_%s', p_table_name, to_char(v_start, 'YYYY_MM'));
        IF NOT EXISTS (SELECT 1 FROM pg_class WHERE relname = v_name) THEN
            EXECUTE format(
                'CREATE TABLE %I PARTITION OF %I FOR VALUES FROM (%L) TO (%L)',
                v_name, p_table_name, v_start, v_end
            );
            RAISE NOTICE 'Created partition: %', v_name;
        END IF;
    END LOOP;
END;
$$;
-- Ejecutar mensualmente vía pg_cron o K8s CronJob
CALL create_monthly_partitions('inbox_events', 2);
CALL create_monthly_partitions('outbox_events', 2);
Para garantizar la operatividad y evitar fallos por falta de segmentos, se recomienda pre-crear las particiones de inbox_events y outbox_events para un horizonte de 24 a 36 meses. Esto asegura una ingesta continua y simplifica el mantenimiento preventivo del Event Store a largo plazo, ahorrandonos el procedure anterior.

8.3. Archivado y Cleanup
Se recomienda el uso de DETACH PARTITION en lugar de DROP para evitar que el usuario del aplicativo requiera permisos elevados de administración. Este enfoque mitiga riesgos de seguridad y permite un archivado o purgado posterior controlado por el DBA, manteniendo el principio de mínimo privilegio.



CREATE OR REPLACE PROCEDURE detach_old_partitions(
    p_table_name       TEXT,
    p_retention_months INT DEFAULT 3
)
LANGUAGE plpgsql AS $$
DECLARE
    v_cutoff DATE;
    v_rec    RECORD;
BEGIN
    v_cutoff := date_trunc('month', now()) - (p_retention_months || ' months')::INTERVAL;
    FOR v_rec IN
        SELECT inhrelid::regclass::text AS partition_name
        FROM pg_inherits
        JOIN pg_class c ON c.oid = inhrelid
        WHERE inhparent = p_table_name::regclass
    LOOP
        IF v_rec.partition_name ~ ('_(\d{4}_\d{2})$') THEN
            DECLARE v_part_date DATE;
            BEGIN
                v_part_date := to_date(
                    substring(v_rec.partition_name FROM '_(\d{4}_\d{2})$'), 'YYYY_MM'
                );
                IF v_part_date < v_cutoff THEN
                    EXECUTE format(
                        'ALTER TABLE %I DETACH PARTITION %I',
                        p_table_name, v_rec.partition_name
                    );
                    RAISE NOTICE 'Detached: %', v_rec.partition_name;
                    -- Siguiente paso: DROP TABLE o pg_dump a cold storage
                END IF;
            END;
        END IF;
    END LOOP;
END;
$$;
8.4. Partial Indexes en el Read Model
Índice parcial sobre WHERE finalized = false (o equivalente por status) en las tablas operativas del Read Model para mantener performance en bultos activos. Los bultos terminales quedan fuera del índice operativo. Ver ix_shipments_status en 7.1.

9. Patrones de Consistencia y Resiliencia
9.1. Idempotencia
Cada evento entrante tiene una clave compuesta (carrier_code, external_event_id) como anchor de idempotencia. El constraint único en el inbox detecta y descarta duplicados antes de que lleguen al worker.

9.2. Sharding de Colas por tracking_id
Para garantizar procesamiento secuencial de eventos de un mismo bulto, se deberia utilizar el tracking_id como sharding key en las colas para asegurar que un mismo worker procese todos los eventos de un bulto en orden (modelo de Actores, lógico). Sin embargo no sabemos si es posible setear sharding key en Fenix.

9.3. Consistencia Eventual
El delay entre escritura en el Write Model y su reflejo en el Read Model es aceptable. El sistema no requiere consistencia fuerte entre servicios.

10. Roadmap MVP (WIP)
Objetivo: Establecer el "contrato social" entre el sistema de datos y los consumidores, garantizando que el Read Model sea funcional y resiliente antes de la integración total con servicios upstream.

Arquitectura Monolítica Modular (NestJS):  Desarrollo de un único servicio con dos módulos internos (Ingestor y DeliveryEngine) compartiendo una instancia de RDS.

Uso de Prisma ORM para establecer los modelos y administrar migrations

Desacople de Contratos (API First):

Disponibilización de los endpoints de Search y Timeline con esquemas finales.

Uso de Swagger/OpenAPI para que los equipos de Frontend (Delivery Manager) consuman mocks dinámicos o datos pre-procesados, permitiendo el desarrollo paralelo de la UI.

Stress Testing & Baseline:

Simulación de ingesta masiva (carga de HotSale) para medir el comportamiento de los B-tree Indexes bajo carga simulada.

Identificación de cuellos de botella en las Closure Tables con volúmenes de datos reales (millones de eventos).

Pre-creación de Particiones:

Automatización de la creación de particiones para los próximos 24 a 36 meses en inbox_events y outbox_events para asegurar ingesta continua.

11. Decisiones Abiertas
Decisión

Opciones

Estado

Mecanismo de claim del worker

Optimistic Lock (version) vs FOR UPDATE SKIP LOCKED

Pendiente (ver 6.4)

Escenario de inbox para V1

Escenario A (estandarizado) vs Escenario B (raw) vs ambos

Pendiente según fuentes disponibles en Fase 1

Mapeo Event Time por operador

Identificar campo de timestamp real por cada carrier

Pendiente por operador o servicio upstream

12. Glosario
Término

Definición

Closure Table

Técnica SQL para almacenar jerarquías de splits de forma consultable sin recursión, en O(1).

Lateness

Tiempo entre el Event Time y el Processing Time de un evento.

Outbox Pattern

Asegura que el cambio en DB y el envío a la cola ocurran de forma atómica.

Processing Time

Momento en que el evento llegó al sistema.

Watermark

Umbral temporal para decidir cuándo cerrar una ventana de consolidación de datos.



sequenceDiagram
    participant ADAPTER as ADAPTER<br/>(Carrier Events)
    participant FQ as Fenix Queue
    participant IC as Ingestor Controller
    participant IUC as IngestEvent UseCase
    participant INBOX as inbox_events
    participant WORKER as Inbox Worker<br/>(Polling Loop)
    participant RM as Read Model<br/>(shipments, tracking_details, status_logs)
    participant OUTBOX as outbox_events
    participant RELAY as Outbox Relay
    participant FQ2 as Fenix Queue<br/>(Outbound)
    participant DHC as DeliveryHistory Controller
    participant SUC as SearchShipments UseCase
    Note over ADAPTER,INBOX: Ingesta Atómica (RFC §6.1)
    ADAPTER->>FQ: Tracking event (carrier webhook)
    FQ->>IC: POST /ingestor/fenix-queue/consume
    IC->>IUC: execute(IngestEventInputDto)
    IUC->>INBOX: INSERT (status=PENDING)
    INBOX-->>IUC: saved entity
    IUC-->>IC: { id, status: PENDING }
    Note over WORKER,OUTBOX: Worker de Procesamiento (RFC §6.4/§6.5)
    loop Polling con Batching
        WORKER->>INBOX: SELECT ... WHERE status=PENDING<br/>LIMIT batch_size FOR UPDATE SKIP LOCKED
        INBOX-->>WORKER: batch de eventos
        WORKER->>INBOX: UPDATE status=PROCESSING,<br/>locked_by=pod_fqdn
        rect rgb(240, 248, 255)
            Note over WORKER,OUTBOX: BEGIN Transaction
            WORKER->>RM: UPSERT shipment<br/>(carrier_code + tracking_number)
            WORKER->>RM: INSERT tracking_detail<br/>(idempotent, ON CONFLICT DO NOTHING)
            WORKER->>RM: INSERT status_log<br/>(audit trail)
            WORKER->>OUTBOX: INSERT outbox_event<br/>(status=PENDING)
            WORKER->>INBOX: UPDATE status=PROCESSED
            Note over WORKER,OUTBOX: COMMIT
        end
    end
    Note over WORKER,INBOX: Error Path
    WORKER--xINBOX: ON ERROR: retry_count++,<br/>status=PENDING o FAILED si max_retries
    Note over RELAY,FQ2: Outbox Relay (Async)
    loop Relay Polling
        RELAY->>OUTBOX: SELECT ... WHERE status=PENDING
        OUTBOX-->>RELAY: batch de eventos
        RELAY->>FQ2: Emit shipment.status_changed
        RELAY->>OUTBOX: UPDATE status=PROCESSED
    end
    Note over DHC,SUC: Búsqueda (Read Model)
    DHC->>SUC: GET /delivery-history/search?query=term
    SUC->>RM: UNION ALL prefix search<br/>(tracking, order_id, external_order_id, customer_id)
    RM-->>SUC: results[]
    SUC-->>DHC: { results, total }
