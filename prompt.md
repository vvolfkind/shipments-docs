# Prompt — Onboarding (delivery-history)

Onboarding mínimo. **Status y pendientes:** solo **`plan.md` § Pendiente**.

---

## Reglas críticas

1. No modificar `AGENTS.md` ni `<rootDir>/framework` — excepción: capa Fenix outbox API (`plan.md` § B.7).
2. Patrón repository + mapper + provider; TypeScript estricto (sin `any`).
3. **NUNCA** editar mocks a mano — regenerar con scripts del proyecto.
4. Tests en fase 2 salvo pedido explícito.

Proyectos: `shipments-history-api/`, `shipments-history-projector/`, `fenix-local/`.

---

## Runtime y MCP (CRÍTICO)

| Acción | Agente | Usuario |
|--------|--------|---------|
| `start:dev`, Docker, build, test, lint, orchestrator, Artillery | **NUNCA** | Sí |
| Editar código, leer docs | Sí | — |
| SQL read-only (MCP `shipments-history-postgres`) | Sí, acotado | — |

MCP: Postgres multidb `localhost:5432` — `shipments-history-api` | `shipments-history-projector`. Una DB por turno; sin cross-DB. Queries con `LIMIT` / por `id` (dataset grande). Validación SQL: **`local-testing-guide.md`**.

---

## Mapa mínimo

| Pregunta | Doc |
|----------|-----|
| ¿Qué falta? | **`plan.md` § Pendiente** |
| Dominio | `domain-model.md` |
| Andreani prod wire samples | `andreaniEvents.md` |
| E2E / SQL validación | `local-testing-guide.md` |
| Orchestrator | `testing-orchestrator/README.md` |
| Spec blocks | `blocks-api.md` / `blocks-projector.md` (diseño; status en `plan.md`) |
| Search perf | `search-performance-spec.md` |

**Validation evidence:** reproducible runs under `testing-orchestrator/runs/` — see `plan.md` § Validation evidence. Do not cite another machine's run IDs; re-run after material changes.

Contratos FE — entregados a equipos; no en repo.

---

## Arquitectura local

| Servicio | Puerto |
|----------|--------|
| fenix-local | `9000` |
| API | `9100` |
| projector | `9200` |

API → outbox → fenix-local → projector (`PACKAGE_NEW_MILESTONE`). Detalle: `local-testing-guide.md` §1.

---

## Handoff (nuevo agente)

1. `prompt.md` — reglas
2. `plan.md` § Pendiente
3. `domain-model.md` si tocás dominio
4. `AGENTS.md` del proyecto target

**Outbox:** gate `return { event }` → interceptor → `FenixOutboxProducer`. Use case no escribe outbox.

---

## Autoridad

1. `domain-model.md` — dominio
2. `plan.md` — status
3. `blocks-*.md` — diseño
4. Este prompt — onboarding solo
