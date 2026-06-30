# Andreani production wire samples

**Purpose:** curated real payloads from `andreani-events` (2026-06-27). **Not exhaustive** — production has thousands of events; these were selected as relevant shapes.

**Authority for mapping rules:** `domain-model.md` §4.4 · `delivery-status-manager.status-mapping.json` · `plan.md` decision log D-08–D-13.

**Timestamps:** `fechaHora` = business event time at Andreani (suffix `Z` = UTC). Redpanda queue timestamp ≈ ingest time; delta ≈ pipeline latency.

---

## Sample index (this file)

| # | `evento` | Mapped? | Notes |
|---|----------|---------|-------|
| 1 | `Visita` | No | `motivoDesc` only — `"No responde llamado"` |
| 2 | `EnvioEntregado` | Yes | `motivo` 99 → `DELIVERED` |
| 3 | `ExpedicionHojaDeRutaDeViaje` | No | `PROCESSED` only — operational noise |
| 4 | `Visita` | **Open (B.5.11)** | desc `"Entregado"` without `motivo` 99 |
| 5 | `Visita` | Yes | `26,22` → failed delivery + context |
| 6 | `Distribucion` | Yes | checkpoint |
| 7 | `Visita` | **Open (B.5.11)** | desc `"Entregado"` |
| 8 | `EnvioEntregado` | Yes | empty `motivo` — key `EnvioEntregado` |
| 9 | `EnvioConsolidado` | Yes | checkpoint |
| 10 | `Visita` | **Open (B.5.11)** | desc `"Entregado"` |

---

## Raw payloads

```json
{
   "idAndreani": "400400005704439",
   "idCliente": "",
   "evento": "Visita",
   "motivo": "",
   "motivoDesc": "No responde llamado",
   "subMotivo": "",
   "subMotivoDesc": "",
   "fechaHora": "2026-06-27T13:05:09Z"
}

{
   "idAndreani": "360003003654660",
   "idCliente": "",
   "evento": "EnvioEntregado",
   "motivo": "99",
   "motivoDesc": "Entregado",
   "subMotivo": "",
   "subMotivoDesc": "",
   "fechaHora": "2026-06-27T13:07:13Z"
}

{
   "idAndreani": "360003010252570",
   "idCliente": "",
   "evento": "ExpedicionHojaDeRutaDeViaje",
   "motivo": "",
   "motivoDesc": "*",
   "subMotivo": "",
   "subMotivoDesc": "*",
   "fechaHora": "2026-06-27T13:08:42.427Z"
}

{
   "idAndreani": "400400005704434",
   "idCliente": "",
   "evento": "Visita",
   "motivo": "",
   "motivoDesc": "Entregado",
   "subMotivo": "",
   "subMotivoDesc": "Entregado",
   "fechaHora": "2026-06-27T13:08:18Z"
}
{
   "idAndreani": "360003016479210",
   "idCliente": "128574593",
   "evento": "Visita",
   "motivo": "26",
   "motivoDesc": "Direccion Incorrecta",
   "subMotivo": "22",
   "subMotivoDesc": "No Existe Calle",
   "fechaHora": "2026-06-27T13:03:45Z"
}
{
   "idAndreani": "400400005679205",
   "idCliente": "",
   "evento": "Distribucion",
   "motivo": "",
   "motivoDesc": "*",
   "subMotivo": "",
   "subMotivoDesc": "*",
   "fechaHora": "2026-06-27T13:01:06.738Z"
}
{
   "idAndreani": "400400005704463",
   "idCliente": "",
   "evento": "Visita",
   "motivo": "",
   "motivoDesc": "Entregado",
   "subMotivo": "",
   "subMotivoDesc": "Entregado",
   "fechaHora": "2026-06-27T13:06:29Z"
}
{
   "idAndreani": "400400005704520",
   "idCliente": "",
   "evento": "EnvioEntregado",
   "motivo": "",
   "motivoDesc": "",
   "subMotivo": "",
   "subMotivoDesc": "",
   "fechaHora": "2026-06-27T12:23:46Z"
}
{
   "idAndreani": "360003014040710",
   "idCliente": "",
   "evento": "EnvioConsolidado",
   "motivo": "",
   "motivoDesc": "*",
   "subMotivo": "",
   "subMotivoDesc": "*",
   "fechaHora": "2026-06-27T13:08:57.9Z"
}
{
   "idAndreani": "400400005572298",
   "idCliente": "",
   "evento": "Visita",
   "motivo": "",
   "motivoDesc": "Entregado",
   "subMotivo": "",
   "subMotivoDesc": "Entregado",
   "fechaHora": "2026-06-27T13:06:11Z"
}
```
