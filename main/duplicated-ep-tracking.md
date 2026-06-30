# duplicated-ep-tracking

## Problema

`ep-ready-to-collect-orders` hoy usa `trackingNumber` de EnvioPack como ancla de identidad temprana. Sabemos que ese `trackingNumber` (`EP_BASE`) puede reutilizarse para órdenes distintas.

Validación importante: `receivedDate` no viene de EnvioPack como event time original del proveedor. Lo genera `ep-collect-webhook` al emitir `deliveryToCollect`, usando `time.Now().UTC()` en `ep-collect-webhook/cmd/api/handlers/notify_delivery.go`.

Eso cambia el análisis:

- no estamos frente a un proveedor que nos reenvía el mismo `occurredAt`
- estamos frente a un emisor propio que asigna un timestamp local al momento de cada notificación
- por lo tanto, `EP_BASE + receivedDateEpochMs` pasa a ser una candidata razonable para identificar de forma única cada draft temprano

El problema actual sigue siendo que, si usamos `EP_BASE` como `Package.externalId` único, distintos anuncios de ready colapsan sobre un mismo package y contaminan `readyToCollectAt`, `PackageShipment` y la lectura histórica.

## Objetivo

Aceptar drafts duplicados de `EP_BASE` con el menor cambio posible, preservando historia y evitando por ahora una redefinición completa de la identidad canónica materializada.

Regla funcional mínima:

- cada `EP_READY` conserva su evidencia propia
- los drafts no proyectan
- cada draft temprano debe ser único aunque `EP_BASE` se reutilice
- al materializar luego un package, `readyToCollectAt` debe poder venir del último draft abierto de ese `EP_BASE` como mínimo
- los drafts viejos no se borran ni se reescriben
- si `classification` llega antes que `journey`, debe poder resolver contra el último draft conocido
- si no existe draft previo al momento de `classification`, se crea uno sintético con epoch de clasificación
- estos casos anómalos no deben contaminar los KPI normales de ready-to-collect → warehouse

## Alcance de esta spec

Esta spec cubre solo la identidad temprana de drafts creados por `EP_READY`.

Queda explícitamente fuera de este turno resolver qué implica cambiar la identidad o la unicidad histórica de `EP_FULL`. Ese relevamiento se hará después.

También queda fuera expandir el orchestrator con demasiada lógica específica para estos edge cases. La intención es cubrir un caso mínimo reproducible, no replicar toda la complejidad de resolución anómala dentro del pipeline de validación.

## Propuesta

## 1. Key del draft temprano

Cada `EP_READY` crea siempre un draft nuevo.

La key del draft se construye con:

```text
EP_BASE + receivedDateEpochMs
```

Formato sugerido para `Package.externalId`:

```text
draft:ep-ready:<trackingNumber>:<receivedDateEpochMs>
```

Semántica:

- `trackingNumber` = `EP_BASE`
- `receivedDateEpochMs` = `receivedDate` convertido a epoch en milisegundos
- esa combinación es única para el draft temprano

Como `receivedDate` lo genera `ep-collect-webhook`, esta key no depende de un timestamp legado del proveedor.

## 2. Búsqueda por `EP_BASE`

Para reconciliar luego vía LPC, el sistema necesita buscar drafts por `EP_BASE` sin escanear toda la tabla.

Por eso:

- `EP_BASE` debe ser searchable e indexado
- `EP_BASE` no debe ser unique
- la unicidad está en la key completa del draft, no en el valor base

La spec no fija todavía si esto se resuelve con:

- una columna dedicada indexada en `packages`
- una proyección equivalente indexada
- otra variante equivalente que evite full scan

Lo que sí fija es la semántica:

- se consulta por `EP_BASE`
- se ordena por `receivedDateEpochMs DESC`
- se elige el draft abierto más nuevo

## 3. `EP_READY` deja de crear packages anclados en `EP_BASE`

`consumeEpReadyToCollectOrders.useCase.ts` deja de buscar por `findByExternalId(trackingNumber)`.

Nuevo comportamiento:

- cada `EP_READY` crea siempre un draft nuevo
- `Package.externalId` pasa a ser la draft key sintética
- `EP_BASE` se persiste como dato searchable no unique
- `metadata.draftState` arranca en `OPEN`
- `PackageEvent` sigue siendo append-only
- `PackageMilestones.readyToCollectAt` se setea con `receivedDate`

## 4. El draft no crea `EP_BASE` como `PackageIdentifierLink`

Este punto sigue siendo clave:

- hoy `package_identifier_links(identifier, identifierType, source)` es único
- si cada draft guardara `EP_BASE` como link, volveríamos a bloquear duplicados

Por eso:

- el draft no debe usar `PackageIdentifierLink` para `EP_BASE`
- `EP_BASE` vive como dato searchable no unique del draft
- el link `EP_BASE` queda reservado para la materialización canónica, si sigue siendo necesario en el diseño posterior

## 5. Materialización posterior por LPC

Cuando `processJourneyReceiptChunk` recibe un `receiptItem` con `groupId = EP_BASE`:

- busca drafts por `EP_BASE`
- filtra drafts `OPEN`
- ordena por `receivedDateEpochMs DESC`
- toma el más nuevo como `selectedDraft`

Si existe `selectedDraft`:

- promueve su `readyToCollectAt` al package materializado si ese package aún no lo tiene
- guarda referencia de trazabilidad al draft promovido
- marca `selectedDraft` como `PROMOTED`
- marca drafts abiertos anteriores del mismo `EP_BASE` como `SUPERSEDED`

Esta spec no redefine todavía cómo debe resolverse `EP_FULL` si existe una colisión histórica posterior. Solo asegura que, cuando aparezca la materialización, el sistema puede recuperar el draft correcto más reciente.

## 5.b Fallback por `classification`

Si `classification` aparece antes que `journey`:

- se intenta asociar el evento al último draft existente del mismo flujo lógico
- si existe un draft elegible, ese draft pasa a ser la base de trazabilidad temprana para el package materializado
- si no existe draft previo, se crea un draft sintético con key basada en:

```text
EP_BASE derivado + classificationEventTimestampEpochMs
```

Como `classification` hoy trae `packageExternalId` y no `groupId`, este fallback asume derivación por convención desde `EP_FULL` hacia `EP_BASE` mientras no exista una fuente mejor.

Objetivo del fallback:

- no perder completamente la trazabilidad operativa
- no bloquear la materialización tardía
- no forzar ahora una redefinición completa de la identidad canónica

Límite explícito:

- este fallback no convierte el caso en confiable para KPI de first mile
- solo evita perder historia operativa y permite reconstrucción “loose” posterior

## 6. Cierre de drafts

Después de materializar el package a partir del draft elegido:

- `selectedDraft` queda con `metadata.draftState = PROMOTED`
- `metadata.promotedToPackageId = <materializedPackageId>`
- los demás drafts `OPEN` del mismo `EP_BASE` quedan con `metadata.draftState = SUPERSEDED`
- `metadata.supersededByPackageId = <materializedPackageId>`

Importante:

- no se borran
- no se fusionan
- no se les reescribe `readyToCollectAt`
- siguen sirviendo como historia operativa

## 6.b Política de KPI y timeline para casos anómalos

Los casos donde EnvioPack reutiliza `EP_BASE` se consideran anómalos y minoritarios.

Regla propuesta:

- se persisten siempre
- se mantienen visibles en timeline operativa
- pueden reconstruirse de forma “loose” desde `classification` y/o `dispatch` cuando el tramo temprano ya no es confiable
- no deben usarse para KPI normales de `readyToCollectAt -> oeLaunchAt`

En práctica:

- si el caso quedó ambiguo en first mile, el projector puede armar la timeline a partir de `classification` o de `dispatch`
- la métrica SLA normal debe excluir estos casos para no mezclar comportamiento estándar con outliers por tracking reutilizado

La intención no es esconder el caso, sino separarlo de la medición de operación normal.

## 7. `Shipment FIRST_MILE` en drafts

Se mantiene la recomendación de no crear identidad logística reusable desde el draft temprano.

Cambio propuesto:

- `consumeEpReadyToCollectOrders.useCase.ts` deja de llamar `ensureFirstMileShipmentLink`

Motivo:

- hoy `shipments(sourceSystem, carrierCode, trackingNumber)` también es único
- si EnvioPack reutiliza `EP_BASE`, compartiríamos un mismo `Shipment` entre bultos distintos
- como el draft no proyecta ni es identidad canónica, esa relación temprana agrega más contaminación que valor

## Invariantes resultantes

- puede haber N drafts con el mismo `EP_BASE`
- cada draft temprano tiene una key única basada en `EP_BASE + receivedDateEpochMs`
- `EP_BASE` es indexado pero no unique
- solo se promociona el draft `OPEN` más nuevo del mismo `EP_BASE`
- los drafts huérfanos no proyectan y no cuentan en vistas

## Cambios en código esperados

## API

### `consumeEpReadyToCollectOrders.useCase.ts`

- eliminar búsqueda por `findByExternalId(trackingNumber)`
- crear siempre draft nuevo
- usar `EP_BASE + receivedDateEpochMs` como key del draft
- dejar de persistir `PackageIdentifierLink` `EP_BASE` en el draft
- dejar de crear `Shipment FIRST_MILE` desde `EP_READY`
- mantener `InboxEvent`, `PackageEvent` y `PackageMilestones.readyToCollectAt`

### `processJourneyReceiptChunk.useCase.ts`

- buscar drafts por `EP_BASE`
- elegir el draft `OPEN` más nuevo por `receivedDateEpochMs`
- promover `readyToCollectAt` desde ese draft al package materializado si aún no existe
- marcar drafts como `PROMOTED` / `SUPERSEDED`

### `consumeWmsPackageClassification.useCase.ts`

- si existe draft elegible previo, asociarlo al flujo materializado
- si no existe draft previo, crear draft sintético con epoch de clasificación
- marcar el caso para exclusión de KPI normales cuando la trazabilidad temprana ya no sea confiable

### Repositorios

Alcance mínimo esperado:

- una consulta indexada para buscar drafts por `EP_BASE`
- una forma de ordenar por `receivedDateEpochMs DESC`
- una forma de actualizar metadata/estado de drafts resueltos

No hace falta helper nuevo compartido.

## Cambios en seeders

## 1. `seedEpReadyToCollectOrders.js`

Debe poder generar explícitamente el caso de tracking duplicado.

Agregar modo opcional:

- `--scenario=duplicated-tracking`

Parámetros sugeridos:

- `--tracking-number=<EP_BASE>` para fijar el tracking reutilizado
- `--duplicates=<n>` para cuántos drafts generar con el mismo `EP_BASE`
- `--minutes-apart=<n>` para escalonar `receivedDate`
- `--count=<n>` sigue disponible para el modo normal

Comportamiento en `duplicated-tracking`:

- enviar `n` eventos `EP_READY` con el mismo `trackingNumber`
- cada evento usa `receivedDate` diferente
- cada evento representa un draft distinto porque la identidad temprana depende de `EP_BASE + receivedDateEpochMs`
- imprimir resumen con:
  - `trackingNumber`
  - timestamps generados
  - último draft esperado para promoción

Ejemplo:

```bash
node seedEpReadyToCollectOrders.js \
  --scenario=duplicated-tracking \
  --tracking-number=EP123456789N \
  --duplicates=3 \
  --minutes-apart=15
```

## 2. `seedJourneyAnnouncementsFromEpReady.ts`

Debe cambiar a:

- buscar drafts por `EP_BASE`
- ordenar por `receivedDateEpochMs DESC`
- para el escenario duplicado, tomar el draft más nuevo
- generar el `receiptItem` con:
  - `groupId = EP_BASE`
  - `packageExternalId = EP_BASE-01`
  - `lastMileId` según el caso del escenario

El seeder no debe promover drafts. Solo debe preparar el input que hará que `processJourneyReceiptChunk` resuelva el caso.

## 3. `seedJourneys.ts`

No necesita cambio para esta primera versión.

## Cambios en orchestrator

## Objetivo

Agregar un escenario reproducible que cubra el caso sin inventar un pipeline nuevo.

## Propuesta

Incorporar un modo de escenario opcional, por ejemplo:

- `SCENARIO_VARIANT=duplicated-ep-tracking`

o un script/npm específico que ejecute:

1. `seedEpReadyToCollectOrders.js --scenario=duplicated-tracking`
2. `seedJourneyAnnouncementsFromEpReady.ts` preparado para ese `EP_BASE`
3. steps existentes del orchestrator:
   - `scenario`
   - `reconcile`
   - `oe-launch`
   - `classification`
   - `dispatch`
   - opcionalmente `andreani`

No hace falta agregar un step de negocio nuevo si el caso queda sembrado antes del `reconcile`.

Restricción importante:

- no conviene cargar el orchestrator con múltiples variantes de resolución anómala en esta etapa
- el objetivo es cubrir un escenario representativo y chico
- los edge cases de reconstrucción tardía deben validarse con asserts acotados, no con un pipeline paralelo completo

## Assertions nuevas del orchestrator

En el escenario duplicado, además de los asserts actuales, validar:

- existen múltiples drafts para el mismo `EP_BASE`
- el draft promovido es el más nuevo por `receivedDate`
- el package materializado recibe `readyToCollectAt` del último draft del mismo `EP_BASE`
- los drafts anteriores quedan `SUPERSEDED`
- los drafts no proyectan
- los drafts resueltos conservan sus propios `PackageEvent` e `InboxEvent`

Para el fallback por `classification`, si se decide cubrirlo en orchestrator, mantenerlo mínimo:

- classification llega sin journey previo
- se crea o reasocia draft sintético
- el caso queda marcado como no apto para KPI normales

No sumar por ahora variantes complejas de `EP_FULL` reutilizado al orchestrator. Ese análisis queda para el turno posterior.

## Dataset mínimo recomendado

Caso base reproducible:

- Draft 1
  - `trackingNumber = EP123456789N`
  - `receivedDate = T0`
- Draft 2
  - mismo `trackingNumber`
  - `receivedDate = T0 + 15m`
- Draft 3
  - mismo `trackingNumber`
  - `receivedDate = T0 + 30m`
- Journey receipt item posterior
  - `groupId = EP123456789N`
  - `packageExternalId = EP123456789N-01`
  - `lastMileId = 3600000000000001`

Resultado esperado:

- existen tres drafts distintos del mismo `EP_BASE`
- se promociona el draft de `T0 + 30m`
- el package materializado recibe `readyToCollectAt = T0 + 30m`
- draft 3 queda `PROMOTED`
- draft 1 y 2 quedan `SUPERSEDED`
- ningún draft proyectado

Caso mínimo adicional opcional para `classification`:

- no existe journey previo
- llega `classification` para `EP_FULL`
- se crea draft sintético basado en epoch de clasificación o se reasocia al último draft elegible
- el caso queda persistido y visible, pero excluido de KPI normales

## Criterios de aceptación

- el sistema acepta múltiples `EP_READY` con igual `trackingNumber`
- no falla por unicidad en `packages`
- no necesita usar `EP_BASE` como `PackageIdentifierLink` del draft
- `EP_BASE` se puede consultar por índice sin full scan
- `readyToCollectAt` del package materializado sale del draft abierto más nuevo del mismo `EP_BASE`
- la historia de drafts previos queda preservada
- el orchestrator puede reproducir y validar el caso de punta a punta
- los casos ambiguos reconstruidos desde `classification` o `dispatch` no contaminan KPI normales

## Pendiente explícito

Queda fuera de esta spec decidir qué hacer si:

- `classification` aparece antes que `journey`
- `EP_FULL` reaparece en otra instancia histórica
- cambiar `EP_FULL` implica modificar identidad canónica, índices o ownership histórico

De esos puntos, el primero queda ahora cubierto como fallback operativo mínimo. Los otros dos siguen diferidos.

Ese relevamiento se hará en un turno posterior.