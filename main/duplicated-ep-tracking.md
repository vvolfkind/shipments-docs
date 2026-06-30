# duplicated-ep-tracking

## Problema

`ep-ready-to-collect-orders` hoy usa `trackingNumber` de EnvioPack como ancla de identidad temprana. Sabemos que ese `trackingNumber` (`EP_BASE`) puede reutilizarse para órdenes distintas.

Validación importante: `receivedDate` no viene de EnvioPack como event time original del proveedor. Lo genera `ep-collect-webhook` al emitir `deliveryToCollect`, usando `time.Now().UTC()` en `ep-collect-webhook/cmd/api/handlers/notify_delivery.go`.

Eso cambia el análisis:

- no estamos frente a un proveedor que nos reenvía el mismo `occurredAt`
- estamos frente a un emisor propio que asigna un timestamp local al momento de cada notificación
- por lo tanto, `EP_BASE + receivedDateEpochMs` pasa a ser una candidata razonable para identificar de forma única cada draft temprano

El problema actual sigue siendo que, si usamos `EP_BASE` como `Package.externalId` único, distintos anuncios de ready colapsan sobre un mismo package y contaminan `readyToCollectAt`, `PackageShipment` y la lectura histórica.

### Estado real del código hoy (no asumir un solo package)

El sistema ya corre un modelo de **dos packages**, esto cambia el encuadre de §3 y §5:

- `EP_READY` crea/reusa un package con `externalId = EP_BASE` (`consumeEpReadyToCollectOrders.useCase.ts:67`). Es este package el que colapsa ante duplicados: `findByExternalId(trackingNumber)` lo reusa y **sobreescribe** `readyToCollectAt` (`:109`), aunque cada `EP_READY` sí conserva su propio `PackageEvent` (UNIQUE por `externalEventId`).
- `processJourneyReceiptChunk.reconcileReceiptItem` crea **otro** package con `externalId = packageExternalId` (`EP_BASE-01`), y hoy enlaza el draft sólo de forma floja vía `metadata.draftPackageId` (`:170-177`). El identifier link `EP_BASE` queda en el draft porque `ensureIdentifierLink` corta si el identifier ya existe.
- Hoy **nada** copia `readyToCollectAt` del draft al package materializado, y `consumeOeLaunch.useCase.ts:261` sólo emite `readyToCollectAt` si ya está seteado en el package materializado. En la práctica el `readyToCollectAt` de first mile **se pierde en proyección hoy**.

Implicancia: la promoción de §5 no es sólo "preservar historia", es **comportamiento nuevo** que recupera un timestamp hoy perdido. Y el path existente `metadata.draftPackageId` (`reconcileReceiptItem:170-177`) debe **reescribirse/retirarse**, no sólo complementarse: con la key sintética, `findByExternalId(item.groupId)` pasa a devolver `null` y el enlace draft↔materializado actual desaparece en silencio.

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

> **Supuesto externo a validar.** La premisa de que `ep-collect-webhook` estampa `time.Now().UTC()` vive en **otro repo** (`ep-collect-webhook/cmd/api/handlers/notify_delivery.go`), no verificable desde acá. La unicidad de la key depende de que `receivedDate` sea distinto por anuncio: **dos `EP_READY` en el mismo milisegundo seguirían colisionando**. Si ese riesgo es real, agregar un desempate (p. ej. `externalEventId` o un sufijo) a la key.

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

**Trabajo concreto implicado (hoy no existe):**

- No hay columna searchable de `EP_BASE` en `packages`. La opción recomendada es **una columna dedicada indexada no-unique** (p. ej. `ep_base` + índice `ix_packages_ep_base`) vía migración Prisma. Es la pieza de trabajo más grande que la spec deja "abierta".
- El contrato `IPackageRepository` sólo tiene `findByExternalId` / `save` / `update`: hace falta un **método nuevo** (p. ej. `findOpenDraftsByEpBase(epBase)`), no se puede resolver con lo existente.

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

- promueve su `readyToCollectAt` al package materializado si ese package aún no lo tiene (**comportamiento nuevo**: hoy ese timestamp no se promueve y se pierde en proyección — ver "Estado real del código hoy")
- guarda referencia de trazabilidad al draft promovido
- marca `selectedDraft` como `PROMOTED`
- marca drafts abiertos anteriores del mismo `EP_BASE` como `SUPERSEDED`

> **Decisión a registrar — elegir el draft más nuevo:** tomar el `receivedDateEpochMs DESC` es "última intención conocida". Para un `readyToCollectAt` de first mile, el draft **más viejo** sería discutiblemente más correcto (primera vez que estuvo listo). Como estos casos quedan excluidos de KPI igual, se elige el más nuevo, pero es una decisión de negocio que debe quedar en el Decision log de `plan.md`, no implícita.

Esta spec no redefine todavía cómo debe resolverse `EP_FULL` si existe una colisión histórica posterior. Solo asegura que, cuando aparezca la materialización, el sistema puede recuperar el draft correcto más reciente.

> **Límite: la colisión canónica sigue abierta.** Esta spec resuelve la unicidad **sólo en la capa draft**. En materialización, el package materializado sigue intentando `ensureIdentifierLink(pkg.id, EP_BASE, 'EP_BASE')`, y `uq_package_identifier_link` es unique. Si `EP_BASE` se reutiliza para una orden **realmente distinta** más tarde, el segundo package materializado **no** puede tomar el link `EP_BASE` (first-writer-wins; no crashea, pero queda sin link). El problema de unicidad canónica no se resuelve, se posterga (ver Pendiente explícito + §5 `EP_FULL`).

## 5.b Fallback por `classification`

Si `classification` aparece antes que `journey`:

- se intenta asociar el evento al último draft existente del mismo flujo lógico
- si existe un draft elegible, ese draft pasa a ser la base de trazabilidad temprana para el package materializado
- si no existe draft previo, se crea un draft sintético con key basada en:

```text
EP_BASE derivado + classificationEventTimestampEpochMs
```

Como `classification` hoy trae `packageExternalId` y no `groupId`, este fallback asume derivación por convención desde `EP_FULL` hacia `EP_BASE` mientras no exista una fuente mejor.

> **Caveat de la derivación `EP_FULL → EP_BASE` (frágil):** la convención hoy es stripear `-01` (lo confirma el seeder: `packageExternalId = ${epBase}-01`). Pero órdenes multi-bulto reales (`-01`, `-02`, …) colapsarían todas al mismo `EP_BASE` draft. Mientras no haya `groupId` en `classification`, este fallback es **best-effort** y refuerza por qué el caso queda fuera de KPI.

> **Invariante a preservar — `consumeWmsPackageClassification` sigue persist-only.** Hoy es Block 6.4 "persist only, no outbox / no proyección" (`plan.md` §9 Do Not Do). Agregar lookup de draft + creación de draft sintético **no** debe introducir emisión a projector. Mantener esa invariante explícita para que la implementación no emita por accidente.

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

> **Implementación de la marca de exclusión — reusar `reconciliationFlags`.** `consumeWmsPackageClassification` ya escribe `metadata.reconciliationFlags` (`missingJourneyAnnouncement`, `missingOeLaunch`, `:35`). El flag de tracking reutilizado (p. ej. `duplicatedEpBase` / `looseTraceability`) debe sumarse a ese mismo array, no inventar un campo nuevo.
>
> **Quién consume el flag está fuera de scope.** El cálculo de KPI no vive en este repo. Esta spec sólo garantiza que el flag se **produce**; honrarlo en analítica/projector queda fuera de este turno.

## 7. `Shipment FIRST_MILE` en drafts

Se mantiene la recomendación de no crear identidad logística reusable desde el draft temprano.

Cambio propuesto:

- `consumeEpReadyToCollectOrders.useCase.ts` deja de llamar `ensureFirstMileShipmentLink`

Motivo:

- hoy `shipments(sourceSystem, carrierCode, trackingNumber)` también es único
- si EnvioPack reutiliza `EP_BASE`, compartiríamos un mismo `Shipment` entre bultos distintos
- como el draft no proyecta ni es identidad canónica, esa relación temprana agrega más contaminación que valor

> **⚠️ Esto NO es un cambio draft-específico — es global, y toca A.1 (DONE).** Quitar la llamada en `consumeEpReadyToCollectOrders` elimina la creación de `Shipment FIRST_MILE` + `PackageShipment` para **todos** los packages, no sólo los duplicados. Ripple verificado:
> - `applyOeLaunchTrailTransitions` (`shipment.repository.ts:113-121`) hace `UPDATE package_shipments SET exited_at … WHERE trail_type = 'FIRST_MILE'`: sin fila first mile, simplemente actualiza 0 filas (no crashea).
> - `consumeOeLaunch.useCase.ts:214` igual hardcodea `trailTransitions.closeFirstMile` en el payload, así que el projector no se rompe.
> - Resultado: el write-model de la API quedaría **sin leg `FIRST_MILE` para ningún package**. Como `C.4` (`segments[]`) está `NOT STARTED`, es tolerable en v1, pero es un cambio de dominio deliberado (desaparece el tramo first mile) que **debe ir como decisión propia en el Decision log de `plan.md`**, no entrar de contrabando bajo "identidad de drafts".
>
> Alternativa a evaluar si se quiere conservar A.1: mover la creación de `Shipment FIRST_MILE` a la materialización (journey / `OE_LAUNCH`) en vez de eliminarla del todo.

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

- **reescribir/eliminar el path actual `metadata.draftPackageId` (`:170-177`)**: con la key sintética, `findByExternalId(item.groupId)` deja de encontrar el draft
- buscar drafts por `EP_BASE` (método nuevo de repo)
- elegir el draft `OPEN` más nuevo por `receivedDateEpochMs`
- promover `readyToCollectAt` desde ese draft al package materializado si aún no existe (timestamp hoy no promovido)
- marcar drafts como `PROMOTED` / `SUPERSEDED`
- mantener el link `EP_BASE` reservado al package materializado (first-writer-wins; ver límite de colisión canónica en §5)

### `consumeWmsPackageClassification.useCase.ts`

- si existe draft elegible previo, asociarlo al flujo materializado
- si no existe draft previo, crear draft sintético con epoch de clasificación
- marcar el caso para exclusión de KPI normales cuando la trazabilidad temprana ya no sea confiable

### Repositorios

Alcance mínimo esperado:

- **migración Prisma**: columna `ep_base` indexada no-unique en `packages` (o equivalente que evite full scan) — hoy no existe
- **método nuevo en `IPackageRepository`** (p. ej. `findOpenDraftsByEpBase`): el contrato actual sólo tiene `findByExternalId` / `save` / `update`
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