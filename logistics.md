# Modelo de negocio

> **Nota (2026-06-24):** Contratos técnicos Andreani/OOLL: `blocks-api.md` §5.8, `domain-model.md` §1.1 y §4.4, `plan.md`. Contexto de negocio; algunos ejemplos pueden estar desactualizados vs código.

En fravega se maneja el modelo "marketplace", donde multiples "sellers" publican productos en nuestro ecommerce y los usuarios compran. Mismo modelo que mercadolibre.
Para esto, hoy se usa EnvioPack como operador de primera milla.

Este proceso puede ser nombrado como "marketplace" o "sellers".

### Colecta por Envío Pack

Una vez concretada una venta por parte de un Seller, EnvíoPack agrupa los pedidos y realiza la colecta física de los bultos desde los depósitos de cada Seller, transportándolos a sus propios almacenes logísticos.

Hay dos modalidades: home delivery y store pick-up.

Para poder hacer la colecta fisica, el seller tiene que poder empaquetar su bulto y etiquetarlo. Esta etiqueta es la que va a escanearse en todo el proceso logistico y se compone de dos datos clave:

1 - El tracking de enviopack
2 - El tracking del operador de ultima milla

Esto queda fisicamente como dos codigos de barra en la misma etiqueta.

El tracking de enviopack comienza siempre con la sigla EP, y finaliza con un numero de secuencia. Por ejemplo
`EP012933329N` es el identificador principal, pero lo que termina en el codigo de barras es `EP012933329N-01`, `EP012933329N-02`, etc.

El tracking de ultima milla es en general un id interno de fravega que tiene el siguiente formato:
`128-18733100-12991123`
Y a veces una version abreviada donde se elimina el numero del medio y queda 
`12812991123`

Sin embargo, con algunos operadores logisticos (particularmente cuando la modalidad es home delivery), la etiqueta tiene un tracking id propio del operador.
Por ejemplo, cuando trabajamos con andreani como operador de ultima milla, "El tracking del operador de ultima milla" tiene otro formato, y depende del contrato utilizado:
`360002990645690` es un ejemplo de un contrato de "paqueteria". Tambien pueden empezar con "33".
`400400005534448` es un ejemplo de contrato para productos con volumetria que necesita ser manejada de manera diferente, e internamente llamamos "bigger", "big" o "maxi". Tambien pueden empezar con "44".

Hay casos en los que se necesita asegurar que se esta entregando mas de un bulto. El ejemplo clasico es el aire acondicionado split. Para esos casos, andreani maneja un contrato en el que se tiene un tracking id pero ademas se tiene otro tracking id para cada bulto, donde generan la secuencia en los ultimos digitos. Por ejemplo
`440400005512345` va a tener internamente `440400005512350` y `440400005512360`, pero solo envia eventos de notificacion de cambio de estado para `440400005512345`.

## Problema a resolver

El resultado final es poder reconstruir la timeline del envio de un bulto, donde vemos que paso en la primera milla, y que paso en la ultima milla. En una primer etapa nos enfocariamos en el modelo paqueteria donde opera enviopack y andreani, dado que estos son bultos que llegan a nuestro warehouse, y eso nos permitiria construir una timeline de uso interno que nos deje saber exactamente que dia y a que hora se aviso que se tiene listo un bulto, a que hora de que dia se colecto por fravega, a que hora llego al warehouse, a que hora se indujo al sorter (si es sorteable, puede no serlo), a que hora se clasifico a un contenedor (y por quien, y que contenedor), y a que hora se despacho del deposito de fravega al deposito de andreani o a una sucursal.

### Notificación de pedidos por Envío Pack

EnvíoPack informa a nuestro TMS los pedidos, dado que tiene potestad sobre su identificador de tracking, pero necesita obtener el segundo, ya sea HD o SPU. 

En este punto, Envío Pack selecciona a Correo Frávega como operador logístico (OOLL) para cualquier modalidad. Esto quiere decir que Fravega va a gestionar la ultima milla, pero no que lo va a hacer con flota propia. Si bien tenemos flota propia, tambien tenemos multiples operadores logisticos 3rd party, como los ejemplos mencionados de andreani.

## Confirmación de pedidos colectados

Cuando los bultos ya se encuentran en el depósito de Envío Pack, enviopack informa a TMS que los pedidos están listos para continuar el proceso logístico interno. Estos informes generan un evento a una queue llamada "ep-ready-to-collect-orders", que es bastante escueto en cuanto a la informacion que expone:

```json
{
    "trackingNumber": "EP012885470N",
    "receivedDate": "2026-06-10T13:33:38.872321686Z"
}
```

TMS, al recibir esta información, va a ir acumulandola para generar en UNIGIS el viaje que agrupa todos los pedidos disponibles para recolección.

## Creación de Viaje (Grupo LPC)

Se crea un viaje bajo el grupo LPC (Listo para Colectar), donde Frávega realiza la recolección de los bultos desde el depósito de Envío Pack.

El chofer escanea y registra los bultos mediante la aplicación UNIGIS Mobile. Cuando ese proceso finaliza, TMS informa a WMS que le van a llegar un grupo de bultos especifico, con esta estructura:

```json
{
    "pickupType": "sellers",
    "warehouseId": "180",
    "appointment": {
        "expectedDate": "2025-06-10T13:22:58.078Z",
        "journeyId": "1234",
        "route": "#0004",
        "vehicle": {
            "licensePlate": "ABC123"
        }
    },
    "receiptItems": [
        {
            "id": "EP007381498N-01",
            "deliveryId": "128-1234567-123456",
            "lastMileId": "xxxxxxxxxxxxx",
            "vtexId": "FVG-4327483-01",
            "itemType": "SPU",
            "distributionType": "SPU",
            "price": 2500,
            "weight": "20",
            "volume": {
                "width": "20",
                "length": "40",
                "height": "25"
            },
            "group": {
                "id": "EP007381498N",
                "quantity": 1
            },
            "destination": {
                "warehouseId": "03",
                "OOLL": null
            }
        },
        {
            "id": "EP007381498N-01",
            "deliveryId": "128-1234567-123456",
            "lastMileId": "xxxxxxxxxxxxx",
            "vtexId": "FVG-4327483-01",
            "itemType": "SPU",
            "distributionType": "SPU",
            "price": 2500,
            "weight": "20",
            "volume": {
                "width": "20",
                "length": "40",
                "height": "25"
            },
            "group": {
                "id": "EP007381498N",
                "quantity": 1
            },
            "destination": {
                "warehouseId": null,
                "OOLL": {
                    "courierName": "Andreani"
                }
            }
        }
    ]
}
```

Al finalizar la recolección y regresar al Centro de Distribución (CD), el viaje es marcado como "Realizado". Sin embargo esto tiene un componente temporal importante: Que se marque como realizado no significa que la mercaderia este fisicamente en el CD.

Lo que hace WMS es registrar este anuncio y esperarlo. Esto se genera como una "orden de entrada", y va a esperar a ser "lanzada" cuando efectivamente la mercaderia este fisicamente en el CD.

Aca seria donde tenemos la primer oportunidad de empezar a consolidad ordenes y la trazabilidad por bulto. Podemos enterarnos muy temprano de que un bulto fue colectado por eviopack y este finalizo su procesamiento, y esta esperando a que lo colectemos, con el topico "ep-ready-to-collect-orders", pero ahi no tenemos informacion suficiente de la orden y demas. Adicionalmente, NO TODO LO QUE PROCESA ENVIOPACK viene a CD de fravega. En muchos casos, la gestion de ultima milla se ejecuta por fuera del CD para los bultos con volumetria alta, por una cuestion de espacio fisico. 

Entonces tenemos 3 puntos en el tiempo que podemos resolver pero que solo un porcentaje serviria para el MVP. Deberiamos registrar cada aviso de "ep-ready-to-collect-orders", y luego cuando se anuncia el viaje de colecta, reconciliar la data para entender cuanto paso desde el aviso hasta que efectivamente se planifico y ejecuto la colecta, y luego cuanto tiempo tardo la mercaderia en efectivamente llegar al CD. El warning es que inicialmente, mucho de lo que nos informe "ep-ready-to-collect-orders" no lo vamos a poder reconciliar.

Pero necesitamos persistir esta informacion para despues relacionarla a los viajes de colecta y tener el primer tramo de informacion para el scope de bultos que si llegan al CD.

## Trazabilidad en WMS

Tal cual dijimos, en WMS tenemos el anuncio del viaje de colecta, y la recepcion fisica en si.

El anuncio pasa inicialmente por un sistema llamado WMI, que es un conjunto de microservicios que llaman "interfaces", y basicamente son adapters a sistemas externos. Uno de estos sistemas externos es TMS, y lo que hace TMS es impactar via rest los viajes de colecta con TODOS los bultos en WMI. WMI persiste temporalmente esta informacion en Redis, y emite un evento con un unique id, que va a hacer que los sistemas que necesiten enterarse de la colecta dentro de WMS, vayan a buscar via REST lo que se persistio en Redis bajo ese uuid.

Nosotros hariamos exactamente eso, para poder empezar a reconciliar la informacion que nos fue llegando de "ep-ready-to-collect-orders".

El sistema WMS, internamente tiene un servicio dedicado a gestion de ordenes de entrada (receptions & storage, RAS), y basicamente hace lo mismo, va a WMI a buscar lo que se informo, y genera una orden de entrada por viaje.

Cuando llega fisicamente esa orden de entrada, RAS tiene un frontend en el cual se puede "lanzar" la misma. Eso hace sus cambios de estado internos, y emite un evento.

El topico es `WM2_RS_MAIN_OE_LAUNCH`, y el evento es 

```json
{
    "metadata": {
        "created_at": "2026-06-08T16:37:22.516165251-03:00",
        "producer": "wm2-rs-main",
        "queue": "WM2_RS_MAIN_OE_LAUNCH"
    },
    "body": "{\"namespace\":\"\",\"eventVersion\":1,\"eventName\":\"OE_LAUNCH\",\"eventUser\":\"85\",\"eventUserData\":{\"user\":{\"id\":\"85\",\"firstName\":\"Pedro\",\"lastName\":\"Monsalvo\",\"username\":\"pedro.monsalvo@fravega.com.ar\"},\"event\":{\"source\":\"Fenix\",\"client\":\"Kovix\",\"producer\":\"wm2-rs-main\"}},\"eventTimestamp\":\"2026-06-08T19:37:22.484Z\",\"data\":{\"locationExternalId\":\"htyftyrAEdfGwQVwbhURrJ\",\"entryOrderExternalId\":\"c90c5d7b-8311-4c80-b2aa-b945dff4a10d\"}}"
}
```

`entryOrderExternalId` es lo que nos dice a cual de los viajes de colecta anunciados e impactados en WMI corresponde esa llegada fisica. Por lo que con este evento, tenemos el timestamp exacto de llegada fisica al CD.

### Procesamiento de bultos

dentro del CD, los bultos son clasificados/consolidados por destino en contenedores con identificadores especificos (pallets o cajas con LPN). Toda vez que un bulto es colocado en un contenedor, WMS emite un evento al topico `WM2_PD_MAIN_CLASSIFICATION` (PD es Preparations&Dispatch o PAD).

El evento tiene esta estructura:

```json
{
    "metadata": {
        "created_at": "2026-06-11T04:54:24.04927192-03:00",
        "producer": "wm2-pd-main",
        "queue": "WM2_PD_MAIN_CLASSIFICATION"
    },
    "body": "{\"eventVersion\":1,\"eventName\":\"ASSIGN_ITEM_TO_CONTAINER\",\"eventUser\":\"832\",\"eventTimestamp\":\"2026-06-11T07:54:24.022Z\",\"namespace\":\"classification.sorterClassification.putPackageInContainer\",\"data\":{\"containerId\":\"990000000000049343\",\"packageExternalId\":\"EP012919848N-01\",\"classificationType\":\"SORTABLE\"}}"
}
```

Como podras ver, tengamos 10 o 10mil bultos, nos vamos a ir enterando de todo el historial.

#### Despacho

Una vez se cierran los contenedores, PAD empieza a registrar en un mismo topico dos tipos de eventos. Uno es la asignacion del contenedor a una orden de salida, y el otro es el despacho efectivo de los contenedores y sus bultos.

Este topico es `WM2_PD_MAIN_DISPATCH`.

Este es un evento en el que se "suben" contenedores de sellers a ordenes de salida:

```json
{
    "metadata": {
        "created_at": "2026-06-11T03:13:32.169854394-03:00",
        "producer": "wm2-pd-main",
        "queue": "WM2_PD_MAIN_DISPATCH"
    },
    "body": "{\"eventVersion\":1,\"eventName\":\"DISPATCH_CONTAINER_INTENT\",\"eventUser\":\"system\",\"eventTimestamp\":\"2026-06-11T06:13:32.137Z\",\"namespace\":\"dispatch.sessions.createDispatchSession.addContainersToOrderIntent\",\"data\":{\"externalOrderId\":\"63571\",\"containers\":[\"990000000000048804\",\"990000000000049445\",\"990000000000049750\"]}}"
}
```

Como nosotros ya sabemos en que contenedor se coloco un bulto, cuando recibimos este evento podemos terminar de asociar el mismo a una orden de salida que va a ser despachada del CD, para luego escuchar este evento, en el mismo topico:

```json
{
    "metadata": {
        "created_at": "2026-06-11T05:18:46.655107911-03:00",
        "producer": "wm2-pd-main",
        "queue": "WM2_PD_MAIN_DISPATCH"
    },
    "body": "{\"eventVersion\":1,\"eventName\":\"CONTAINER_PACKAGES_SHIPPED\",\"eventUser\":\"SYSTEM\",\"eventTimestamp\":\"2026-06-11T08:18:46.600Z\",\"namespace\":\"dispatch.expedition.containerPackagesShipped.success\",\"data\":{\"journeyId\":\"62937\",\"routeId\":\"63512\",\"route\":{\"containerIds\":[\"990000000000049502\"],\"origin\":\"180\",\"vehicle\":{\"licensePlate\":\"IUI272\",\"vehicleType\":\"truck\",\"courierName\":\"TRANSPORTES BENJA S.R.L.\"},\"orderItems\":[{\"id\":\"EP012854177N-01\",\"groupId\":\"EP012854177N\",\"deliveryId\":\"128-18625997-12914024\",\"status\":\"SHIPPED\"},{\"id\":\"EP012861399N-01\",\"groupId\":\"EP012861399N\",\"deliveryId\":\"128-18641651-12920914\",\"status\":\"SHIPPED\"},{\"id\":\"EP012864448N-01\",\"groupId\":\"EP012864448N\",\"deliveryId\":\"128-18646529-12923839\",\"status\":\"SHIPPED\"},{\"id\":\"EP012865239N-01\",\"groupId\":\"EP012865239N\",\"deliveryId\":\"128-18648253-12924579\",\"status\":\"SHIPPED\"},{\"id\":\"EP012865838N-01\",\"groupId\":\"EP012865838N\",\"deliveryId\":\"128-18649365-12925144\",\"status\":\"SHIPPED\"},{\"id\":\"EP012866629N-01\",\"groupId\":\"EP012866629N\",\"deliveryId\":\"128-18651019-12925872\",\"status\":\"SHIPPED\"},{\"id\":\"EP012866855N-01\",\"groupId\":\"EP012866855N\",\"deliveryId\":\"128-18651745-12926097\",\"status\":\"SHIPPED\"},{\"id\":\"EP012866875N-01\",\"groupId\":\"EP012866875N\",\"deliveryId\":\"128-18652026-12926117\",\"status\":\"SHIPPED\"},{\"id\":\"EP012867350N-01\",\"groupId\":\"EP012867350N\",\"deliveryId\":\"128-18653634-12926591\",\"status\":\"SHIPPED\"},{\"id\":\"EP012868681N-01\",\"groupId\":\"EP012868681N\",\"deliveryId\":\"128-18655432-12927922\",\"status\":\"SHIPPED\"},{\"id\":\"EP012872770N-01\",\"groupId\":\"EP012872770N\",\"deliveryId\":\"128-18656169-12931931\",\"status\":\"SHIPPED\"},{\"id\":\"EP012872940N-01\",\"groupId\":\"EP012872940N\",\"deliveryId\":\"128-18656154-12932101\",\"status\":\"SHIPPED\"},{\"id\":\"EP012877976N-01\",\"groupId\":\"EP012877976N\",\"deliveryId\":\"128-18656732-12937105\",\"status\":\"SHIPPED\"},{\"id\":\"EP012878379N-01\",\"groupId\":\"EP012878379N\",\"deliveryId\":\"128-18656971-12937492\",\"status\":\"SHIPPED\"},{\"id\":\"EP012878638N-01\",\"groupId\":\"EP012878638N\",\"deliveryId\":\"128-18657232-12937735\",\"status\":\"SHIPPED\"},{\"id\":\"EP012879031N-01\",\"groupId\":\"EP012879031N\",\"deliveryId\":\"128-18658368-12938105\",\"status\":\"SHIPPED\"},{\"id\":\"EP012879286N-01\",\"groupId\":\"EP012879286N\",\"deliveryId\":\"128-18658700-12938356\",\"status\":\"SHIPPED\"},{\"id\":\"EP012879304N-01\",\"groupId\":\"EP012879304N\",\"deliveryId\":\"128-18658831-12938374\",\"status\":\"SHIPPED\"},{\"id\":\"EP012879460N-01\",\"groupId\":\"EP012879460N\",\"deliveryId\":\"128-18659230-12938526\",\"status\":\"SHIPPED\"},{\"id\":\"EP012879753N-01\",\"groupId\":\"EP012879753N\",\"deliveryId\":\"128-18659946-12938809\",\"status\":\"SHIPPED\"},{\"id\":\"EP012880721N-01\",\"groupId\":\"EP012880721N\",\"deliveryId\":\"128-18661761-12939726\",\"status\":\"SHIPPED\"},{\"id\":\"EP012881544N-01\",\"groupId\":\"EP012881544N\",\"deliveryId\":\"128-18663501-12940540\",\"status\":\"SHIPPED\"},{\"id\":\"EP012884408N-01\",\"groupId\":\"EP012884408N\",\"deliveryId\":\"128-18666609-12943300\",\"status\":\"SHIPPED\"},{\"id\":\"EP012890968N-01\",\"groupId\":\"EP012890968N\",\"deliveryId\":\"128-18670609-12949773\",\"status\":\"SHIPPED\"}],\"currentStatus\":\"SHIPPED\"}}}"
}
```

Esto nos permite actualizar un conjunto de bultos, para los cuales generariamos un evento de cambio de estado que haria que pase de "procesando" a "despachado", lo cual podria traducirse a "en transito" o lo que corresponda segun definamos.

## Especificaciones de ordenes de salida WMS

Las ordenes de salida de WMS pueden ser de abastecimiento a sucursales (store supply, SS), o de home delivery (hd). Dado que el modelo de fravega no es solo marketplace, sino tambien retail, con stock propio, lo que termina pasando es que el WMS se usa para obtener la demanda comercial, asignarle stock, y planificar como se va a gestionar todo el circuito.
Al incorporar un modelo que es practicamente cross-docking, lo que termina pasando es que el punto de convergencia es el destino de todo lo que se consolida a contenedores.
Nunca va a existir un contenedor que vaya a sucursal 30 y sucursal 71, o a Andreani y Moova. El contenedor siempre va a tener un unico destino. Esto es asi de siempre, y lo que se hizo fue montar el flujo de marketplace/sellers para aprovechar que esa planificacion ya existe y se esta ejecutando, y lo que termina pasando es que los contenedores de sellers se "suben" a ordenes de salida de retail, pero podemos tener trazabilidad independiente de los bultos que componen las mismas, enfocandonos unicamente en sellers. Adicionalmente, se aprovechan las ordenes de abastecimiento para incorporar el modelo de store pick-up (SPU).

Asi como en el modelo store supply una orden de salida puede ir a una sucursal, o a multiples, pero cada contenedor se baja en su destino, las ordenes de home delivery son un circuito aparte pero que cumple con reglas similares: El destino es el warehouse del operador logistico.

Entonces, para el scope de home delivery, podemos trazar POR bulto, cuando se anuncio que esta disponible, cuando se anuncio que esta en camino al CD, cuando llego fisicamente al CD, cuando se puso en un contenedor (y en que contenedor, y por quien), y cuando se despacho del CD.

## Ultima milla con OOLL

Para cuando se despacha del CD, puede que los bultos con tracking id de andreani ya hayan tenido algun evento, y nosotros lo hayamos consumido. La realidad es que tenemos que reconciliar todo lo que viene despues, registrando cada cambio de estado, lo cual nos va a dar trazabilidad de ultima milla quizas no tan fina como logramos por aprovechar la arquitectura del WMS, pero si para completar la timeline.

Ese consumer de andreani-events ya esta implementado, y antes de pensar en la reconciliacion, tenemos que entender como soportar todo lo que se describio hasta ahora.

---

# Modelo de Entidades Mínimas para Trazabilidad por Bulto (MVP + Escalabilidad)

## Decisión Arquitectónica

La premisa es simple: **TRAZABILIDAD POR BULTO primero, reconciliación con entidades logísticas/comerciales después**.

Esto significa:
- El bulto es la unidad canónica de trazabilidad en el modelo de persistencia.
- Los eventos se registran contra el bulto, independientemente del tramo logístico en que se encuentre.
- Las entidades Order, Delivery, Shipment son contexto comercial/operativo superior, NO la raíz de trazabilidad.
- Las timelines (simple, por tramo, detallada) son vistas/proyecciones del mismo modelo persistente, no tablas paralelas.

## Entidades Mínimas

### Package (Bulto Canónico)

Representa la unidad física trazable. Es la raíz absoluta de la trazabilidad.

```
Package (bulto)
├─ id (UUID)
├─ externalId (VARCHAR 256) — identificador operativo único del bulto: EP012933329N-01, lastMileTrackingId, etc.
├─ sourceSystem (VARCHAR 64) — origen: "ENVIOPACK", "WMS", "RETAIL", etc.
├─ deliveryId (VARCHAR 64) — reconciliación con la entidad comercial (puede null al inicio)
├─ orderId (VARCHAR 64) — reconciliación auxiliar (puede null al inicio)
├─ currentStatus (VARCHAR 32) — estado calculado: "CREATED", "READY_TO_COLLECT", "COLLECTED", "IN_WAREHOUSE", "CLASSIFIED", "DISPATCHED", "IN_TRANSIT", "DELIVERED", etc.
├─ metadata (JSONB) — almacena identificadores heterogéneos: {epBase, epSequence, lastMileId, entryOrderId, containerId, ...}
├─ createdAt, updatedAt
└─ packageEvents (relación 1:N)
```

**Nota:** `externalId` es el atributo clave para deduplicación cross-fuente. Cada evento debe llegar con un identificador único operativo.

### PackageEvent (Append-Only Log)

Cada hito del bulto genera un evento inmutable. Los milestones se derivan de estos eventos.

```
PackageEvent (evento por bulto)
├─ id (UUID)
├─ packageId (FK Package)
├─ source (VARCHAR 64) — sistema que emitió: "ENVIOPACK", "WMS_RS_MAIN", "WMS_PD_MAIN", "ANDREANI", etc.
├─ topic (VARCHAR 128) — tópico de origen: "ep-ready-to-collect-orders", "WM2_RS_MAIN_OE_LAUNCH", "WM2_PD_MAIN_CLASSIFICATION", etc.
├─ eventName (VARCHAR 64) — "EP_READY", "OE_LAUNCH", "ITEM_CLASSIFIED", "CONTAINER_SHIPPED", "STATUS_CHANGED", etc.
├─ occurredAt (TIMESTAMPTZ) — Event Time del operador (no cuando llegó al sistema)
├─ payload (JSONB) — evento raw completo
├─ externalEventId (VARCHAR 256) — deduplicación por fuente
├─ createdAt (TIMESTAMPTZ) — Processing Time (cuándo llegó al sistema)
├─ UNIQUE (packageId, source, externalEventId) — deduplicación robusta
└─ INDEX (packageId, occurredAt DESC) — timeline ordenada por evento
```

### PackageMilestones (Snapshot Opcional)

Para queries rápidas sin reconstruir timeline desde eventos. Contiene **timestamps operativos** (`*At`), no milestones de UI.

> **Definición cerrada:** ver `domain-model.md` §1 (capas), §1.1 (timestamps) y §4.4 (mapeo Andreani). Los `MilestoneKey` de grilla (5 valores: progreso + `DELIVERED` / `FAILED` / `CANCELLED`) viven en projector `current_milestone`, no en esta tabla.

### PackageIdentifierLink (Reconciliación de IDs Heterogéneos)

Tabla de correlación que mapea identificadores operativos dispersos a un Package canónico. Permite llegada desordenada de eventos.

```
PackageIdentifierLink (mapeo N:1 a Package)
├─ id (UUID)
├─ packageId (FK Package)
├─ identifier (VARCHAR 128) — un de: EP012933329N-01, lastMileId, containerId, entryOrderId, externalOrderId, etc.
├─ identifierType (VARCHAR 32) — "EP_FULL", "EP_BASE", "LAST_MILE", "CONTAINER", "ORDER", "ENTRY_ORDER"
├─ source (VARCHAR 64) — sistema que emitió: "ENVIOPACK", "WMS", "ORDER_SYSTEM"
├─ createdAt
└─ UNIQUE (identifier, identifierType, source) — evita duplicados
└─ INDEX (identifier) — lookup rápido "¿a qué Package corresponde este tracking?"
```

### Shipment (Tramo Logístico)

Representa un **tramo** con operador. Modelo binario v1 — ver `domain-model.md` §3:

| `trailType` | Alcance |
|-------------|---------|
| `FIRST_MILE` | Hasta llegada física al CD (`oeLaunchAt`) |
| `LAST_MILE` | Desde `oeLaunchAt` — incluye operativa CD + OOLL + entrega |

`WAREHOUSE_INTERNAL` no se usa como tercer tramo en v1.

```
Shipment (tramo logístico)
├─ id (UUID)
├─ deliveryId (FK Delivery, puede null)
├─ orderId (VARCHAR 64)
├─ trailType (VARCHAR 32) — "FIRST_MILE" | "LAST_MILE"
├─ carrierCode (VARCHAR 32)
├─ trackingNumber (VARCHAR 128)
├─ status, subStatus, canonicalStatus
├─ metadata (JSONB)
├─ packages (N:M via PackageShipment)
└─ createdAt, updatedAt
```

### PackageShipment (Relación N:M)

Vincula bultos con tramos. Un bulto puede pasar por múltiples shipments en su vida.

```
PackageShipment (unión N:M)
├─ packageId (FK Package)
├─ shipmentId (FK Shipment)
├─ enteredAt (TIMESTAMPTZ) — cuándo el bulto entró a este tramo
├─ exitedAt (TIMESTAMPTZ, nullable) — cuándo salió (null si aún está dentro)
└─ PRIMARY KEY (packageId, shipmentId)
```

### Delivery & Order (Sin cambios de estructura)

```
Delivery (raíz de compromiso logístico)
├─ id (UUID)
├─ orderId (FK Order, puede null)
├─ externalOrderId (VARCHAR 64) — reconciliación auxiliar
├─ status, subStatus (derivados de sus packages/shipments)
├─ packages (relación 1:N, para queries rápidas)
└─ ...

Order (nodo comercial)
├─ id (UUID)
├─ externalOrderId (UNIQUE)
├─ customerId
├─ status, subStatus
└─ ...
```

## Eventos y Mapeo a Entidades

### Ejemplo 1: ep-ready-to-collect-orders

```json
{
  "trackingNumber": "EP012933329N",
  "receivedDate": "2026-06-10T13:33:38.872Z"
}
```

**Procesamiento:**
1. Crear o recuperar Package por externalId "EP012933329N".
2. Crear PackageIdentifierLink("EP012933329N", "EP_BASE", "ENVIOPACK").
3. Crear PackageEvent(source="ENVIOPACK", topic="ep-ready-to-collect-orders", eventName="EP_READY", occurredAt=receivedDate, payload=...).
4. Actualizar PackageMilestones.readyToCollectAt = receivedDate.

### Ejemplo 2: OE_LAUNCH (WMS emitió llegada física)

```json
{
  "entryOrderExternalId": "c90c5d7b-8311-4c80-b2aa-b945dff4a10d",
  "packageExternalIds": ["EP012854177N-01", "EP012861399N-01", ...],
  "eventTimestamp": "2026-06-08T19:37:22.484Z"
}
```

**Procesamiento:**
1. Buscar Packages por externalId en packageExternalIds (están indexados).
2. Para cada Package:
   - Crear PackageIdentifierLink(entryOrderExternalId, "ENTRY_ORDER", "WMS").
   - Crear PackageEvent(source="WMS_RS_MAIN", topic="WM2_RS_MAIN_OE_LAUNCH", eventName="OE_LAUNCH", ...payload).
   - Actualizar PackageMilestones.oeLaunchAt = eventTimestamp.
3. Crear o actualizar Shipment(trailType="WAREHOUSE_INTERNAL", carrierCode="FRAVEGA", trackingNumber=null).
4. Crear PackageShipment(packageId, shipmentId, enteredAt=eventTimestamp).

### Ejemplo 3: CLASSIFICATION (WMS asignó a contenedor)

```json
{
  "packageExternalId": "EP012919848N-01",
  "containerId": "990000000000049343",
  "classificationType": "SORTABLE",
  "eventTimestamp": "2026-06-11T07:54:24.022Z"
}
```

**Procesamiento:**
1. Buscar Package por externalId.
2. Crear PackageIdentifierLink(containerId, "CONTAINER", "WMS").
3. Crear PackageEvent(...eventName="ITEM_CLASSIFIED").
4. Actualizar PackageMilestones.classifiedAt = eventTimestamp.

### Ejemplo 4: CONTAINER_PACKAGES_SHIPPED (salida física del CD)

```json
{
  "containerIds": ["990000000000049502"],
  "packageExternalIds": ["EP012854177N-01", ...],
  "eventTimestamp": "2026-06-11T08:18:46.600Z"
}
```

**Procesamiento:**
1. Para cada Package:
   - Crear PackageEvent(...eventName="CONTAINER_SHIPPED", payload=...).
   - Actualizar PackageMilestones.dispatchedAt = eventTimestamp.
   - Cerrar PackageShipment.exitedAt (el bulto sale del tramo interno).
2. Crear nuevo Shipment(trailType="LAST_MILE", carrierCode="ANDREANI", trackingNumber=??).
3. Crear PackageShipment para el nuevo tramo (enteredAt=eventTimestamp).

### Ejemplo 5: Eventos de OOLL (Andreani status change)

```json
{
  "trackingId": "360002990645690",
  "status": "En tránsito",
  "eventTimestamp": "2026-06-12T10:00:00Z"
}
```

**Procesamiento:**
1. Buscar Package por externalId "360002990645690" (lookup en PackageIdentifierLink).
2. Crear PackageEvent(source="ANDREANI", topic="andreani-events", eventName="STATUS_CHANGED", occurredAt=eventTimestamp, payload=...).
3. Actualizar PackageMilestones.inTransitAt (si no estaba ya).

## Escalabilidad

### Retail (Stock Propio)

- Los bultos de retail entran por otro canal (no por EP).
- Se crean Packages con externalId del retail system y sourceSystem="RETAIL".
- El flujo es idéntico: eventos → PackageEvents → timeline.
- Las Shipments pueden ser "RETAIL_SUPPLY" o "STORE_DELIVERY", misma lógica N:M.

### Cambio de Proveedor de Primera Milla

- Si EP se reemplaza por otro proveedor, solo cambia:
  - El identificador del nuevo proveedor (nuevo formato en externalId).
  - La fuente en PackageEvent (source="NEW_PROVIDER").
  - Los mapeos de identificadores en PackageIdentifierLink.
- El modelo no necesita refactor.

### Nuevos Tramos Logísticos

- Cross-dock externo, consolidación regional, etc.
- Se suma un Shipment con trailType nuevo.
- Se vincula via PackageShipment.
- Los eventos llegan al mismo Package append-only log.

### Queries Complejas (Posterior, en Post-Processors)

Ejemplo: "timeline de un bulto vista por cliente".
```sql
SELECT pe.eventName, pe.occurredAt, pe.source
FROM package_events pe
WHERE pe.packageId = $1
ORDER BY pe.occurredAt ASC
```

Ejemplo: "timeline completa con detalles de shipments".
```sql
SELECT 
  pe.eventName, pe.occurredAt, pe.source, s.trailType, s.carrierCode
FROM package_events pe
LEFT JOIN package_shipments ps ON ps.packageId = pe.packageId
  AND pe.occurredAt BETWEEN ps.enteredAt AND COALESCE(ps.exitedAt, NOW())
LEFT JOIN shipments s ON s.id = ps.shipmentId
WHERE pe.packageId = $1
ORDER BY pe.occurredAt ASC
```

## Implementación en API (MVP)

En la API (shipments-history-api), ahora:

1. **Ingestores de eventos**: listeners por tópico (ep-ready-to-collect-orders, WM2_*, andreani-events).
2. **Mappers**: eventPayload → PackageEvent + PackageIdentifierLink + Package.
3. **Repositories**: persistencia atómica en Package + PackageEvent + PackageIdentifierLink.
4. **No post-processors aún**: los milestones se setean manualmente al ingestar, o quedan null para cálculo posterior.

La lógica de post-procesamiento (reconciliación robusta, cálculo de milestones, derivación de estado agregado) queda para la siguiente iteración, cuando la ingesta esté estable y el projector entre en juego.

---

## Building Blocks: Flujo de Reconciliación (orden de implementación)

La idea de esta sección es bajar el enfoque a bloques concretos, en orden lógico, para poder implementar, validar, y avanzar por etapas sin cerrar un modelo rígido al warehouse.

### Principios de diseño (para todo el roadmap)

- Persistir primero, reconciliar después.
- Nada en memoria como fuente de verdad para correlación.
- Reconciliación determinística e idempotente.
- Warehouse-first para MVP, pero con modelo abierto para tramos externos (ejemplo: bulto no pasa por CD y va directo a OOLL).
- No forzar un único circuito: soportar warehouse interno + actores externos + consultas futuras a sistemas auxiliares.

### Block 0 - Baseline de eventos crudos

**Objetivo:** asegurar que todos los eventos relevantes se persisten crudos en inbox.

**Incluye:**
- `ep-ready-to-collect-orders` (ya implementado).
- `andreani-events` (ya implementado).
- `WMI_PICKUP_ROUTE` / `FENIX_QUEUE_WMI_JOURNEY_ANNOUNCEMENT` (en implementación).

**Criterio de validación:**
- Cada mensaje entrante tiene `externalEventId` y queda persistido en `inbox_events`.
- Reprocesar mismo mensaje no duplica por constraint de idempotencia.

### Block 1 - Ingesta del anuncio WMI (fetch por `wmiExternalId`)

**Objetivo:** escuchar evento de anuncio, resolver recurso remoto, y dejar evidencia local para reconciliación posterior.

**Flujo:**
1. Consume evento `WMI_PICKUP_ROUTE`.
2. Extrae `wmiExternalId`.
3. Llama `WMI_API_WEBHOOK_URL/{wmiExternalId}`.
4. Loggea y persiste resultado (aunque no se reconcilie todavía).

**Criterio de validación:**
- Consumer se registra en `FENIX_QUEUE_WMI_JOURNEY_ANNOUNCEMENT`.
- Request al webhook se ejecuta con URL correcta.
- Payload obtenido queda disponible para la siguiente etapa.

### Block 2 - Persistencia normalizada del batch de colecta

**Objetivo:** almacenar el batch WMI de forma consultable, sin depender del payload crudo para todo.

**Nuevas entidades propuestas (API):**
- `PickupJourney`: cabecera del anuncio (`wmiExternalId`, `announcedAt`, `source`, `rawPayload`).
- `PickupJourneyPackage`: detalle por bulto (`packageExternalId`, `deliveryId`, `lastMileId`, `groupId`, `wmiExternalId`).

**Criterio de validación:**
- Un `wmiExternalId` se guarda una sola vez.
- Se puede consultar "qué bultos llegaron en este anuncio" por índice.

### Block 3 - Reconciliación inicial EP_READY -> JourneyPackage

**Objetivo:** vincular avisos tempranos de EP con bultos anunciados por WMI, de manera determinística.

**Reglas v1 (simples):**
- Match exacto por `packageExternalId` / tracking completo cuando exista.
- Si no hay match, queda en `PENDING_MATCH` (no bloquear flujo).

**Tabla de vínculo sugerida:**
- `PackageReconciliationLink`:
    - `packageId`
    - `pickupJourneyPackageId`
    - `matchRule`
    - `matchScore`
    - `status` (`MATCHED`, `PENDING`, `AMBIGUOUS`)

**Criterio de validación:**
- Re-ejecutar reconciliación no duplica links.
- Queda trazabilidad explícita de por qué matcheó (regla y score).

### Block 4 - Milestone de anuncio de viaje y actualización masiva

**Objetivo:** usar el vínculo para actualizar milestones por lote cuando aparezcan hitos de entrada/lanzamiento.

**Ejemplo:**
- Llega OE_LAUNCH con `entryOrderExternalId`.
- Se resuelve viaje/anuncio relacionado.
- Se actualiza `oeLaunchAt` de todos los bultos vinculados en batch.

**Criterio de validación:**
- Update masivo transaccional.
- Total de filas actualizadas coincide con total de bultos relacionados.

### Block 5 - Reconciliación avanzada (sin frenar MVP)

**Objetivo:** sumar reglas de match secundarias para mejorar cobertura.

**Reglas candidatas:**
- `EP_BASE + sequence`.
- `deliveryId` + ventana temporal.
- `lastMileId` + ventana temporal.

**Criterio de validación:**
- Se incrementa `matched_rate` sin subir `ambiguous_rate` fuera de umbral.

### Block 6 - Tramos externos (no warehouse) y timeline completa

**Objetivo:** cubrir casos donde la operación no pisa CD, pero igual necesitamos trazabilidad.

**Caso ejemplo:**
- EnvíoPack colecta heladera.
- Última milla la ejecuta Andreani directo, sin warehouse Frávega.
- Igual debemos reconstruir timeline con eventos externos orquestados por nosotros.

**Estrategia:**
- Mantener `Package` como raíz única.
- Asociar eventos de OOLL al mismo `Package` vía `PackageIdentifierLink`.
- Si existen sistemas externos de viajes/órdenes para enriquecer, consumirlos como fuentes complementarias (no reemplazo de inbox).

**Criterio de validación:**
- Se puede consultar timeline de un bulto aunque no tenga hitos internos de warehouse.
- El modelo soporta ambos caminos: warehouse-heavy y external-heavy.

### Block 7 - Observabilidad y operación

**Objetivo:** medir calidad de reconciliación y detectar deriva temprana.

**Métricas mínimas:**
- `pending_match_count`
- `matched_rate`
- `ambiguous_rate`
- `ep_ready_to_match_latency`
- `match_to_oe_launch_latency`

**Criterio de validación:**
- Dashboards/queries simples para seguimiento diario.
- Alertas cuando sube backlog o cae tasa de match.

## Estado sugerido actual

- Block 0: en curso alto (EP y Andreani listos; WMI announcement en implementación).
- Block 1: en curso.
- Blocks 2 a 7: pendientes por iteración.

## Notas de alcance

- El foco actual sigue siendo warehouse, pero la solución no debe bloquear trazabilidad externa.
- Reconciliación con otros sistemas de viajes/órdenes es evolución natural del modelo, no cambio de paradigma.
- Evitar premisas rígidas de "todo pasa por CD": debe existir camino feliz para ese caso y camino válido para casos externos.





