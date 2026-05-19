# Shutdown watchdogs — Walter WhatsApp

Regla operativa obligatoria: si una campaña se pausa, se elimina, termina, o si un celular queda restringido/baneado, Walter debe desconectar los GeeLark cloud phones sí o sí.

## Disparadores obligatorios

- `campaign_runs.status = paused`
- `campaign_runs.status = completed`
- `campaign_runs.status = failed`
- `campaign_runs.status = cancelled`
- `campaign_runs.status = deleted` o run eliminado desde dashboard
- `campaign_run_device_batches.operational_status = restricted`
- `campaign_run_device_batches.operational_status = banned`
- `devices.operational_status = restricted`
- `devices.operational_status = banned`

## Capas de protección

1. **Dashboard/API**
   - `pause` marca `shutdown_required=true` y dispara shutdown del run.
   - `delete` dispara shutdown antes de borrar datos del run.
   - cambio manual a `restricted`/`banned` dispara shutdown del celular afectado.
   - evento de agente con `analysis_status=restricted|banned` dispara shutdown del celular afectado.

2. **Worker watchdog local**
   - corre cada minuto.
   - si detecta run pausado/finalizado o celulares restringidos/baneados, ejecuta `phone/stop`.
   - si la API no tiene credencial GeeLark o falla, el worker local es el fallback obligatorio.

3. **Triple verificación GeeLark**
   - después de `phone/stop`, consulta `phone/status` repetidamente.
   - exige 3 lecturas consecutivas sin `status=0` antes de declarar OK.
   - si ve un celular abierto en el intento 3, 7 o 10, reintenta `phone/stop`.

## Logs esperados

- `shutdown_watchdog_started`
- `shutdown_watchdog_verified`
- `shutdown_watchdog_failed`
- `shutdown_watchdog_api_fallback_required`
- `restricted_device_disconnected`
- `restricted_device_disconnect_failed`

## Verificación manual rápida

```bash
python3 /root/.hermes/profiles/walter/scripts/walter_campaign_autoworker.py
```

Resultado correcto cuando todo está cerrado:

```text
[AUTO_WORKER] shutdown watchdog: run_status_paused ok=True confirmations=3
```

## Criterio de cierre

No alcanza con que el dashboard diga `paused`. El cierre se considera correcto solo si:

- GeeLark reporta 0 celulares abiertos en el grupo operativo.
- Hay evento auditable de watchdog en Supabase/dashboard.
- El worker queda programado para seguir revisando periódicamente.
