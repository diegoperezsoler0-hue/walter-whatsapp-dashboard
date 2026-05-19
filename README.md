# Walter WhatsApp Dashboard

Dashboard operativo privado para campañas WhatsApp de Toni usando GeeLark, Google Sheets y Supabase.

## Qué hace

- Muestra analytics de campaña: enviados, problemas, celulares activos/restringidos y health.
- Controla runs live: iniciar, pausar, reanudar y eliminar runs.
- Muestra matriz `batch × celular` con estado por teléfono.
- Muestra logs live de eventos del worker.
- Permite editar parámetros operativos: pausas globales, cap por celular, grupo activo, campaña activa, modo dry/live.
- Consume una API Supabase Edge Function protegida con Basic Auth.

## Regla de seguridad

Este repo **no contiene credenciales reales**.

Aunque sea privado, no se versionan:

- Bearer token de GeeLark.
- Supabase service role key.
- Google OAuth tokens.
- Proxy passwords.
- Password real del dashboard.
- URLs remotas de cloud phones con token.

Las credenciales deben cargarse como variables de entorno o secrets del host/Supabase/GitHub. Ver `docs/SECRETS.md` y `.env.example`.

## Arquitectura punta a punta

1. **Google Sheets**
   - Columna A: número destino.
   - Columna B: mensaje exacto.
   - Columna C: estado (`FALSE` pendiente, `enviado` solo después de screenshot verificado).

2. **Supabase/Postgres**
   - Guarda campañas, runs, celulares, eventos y matriz batch×celular.
   - Tablas principales:
     - `app_settings`
     - `campaigns`
     - `campaign_runs`
     - `campaign_run_events`
     - `campaign_run_device_batches`
     - `devices`

3. **Supabase Edge Function**
   - Expone la API JSON usada por el dashboard.
   - Endpoint actual configurado en `app.js`:
     - `https://ezwwbfsdduehtwavmigv.functions.supabase.co/walter-dashboard`
   - Requiere Basic Auth desde el navegador.

4. **Dashboard estático GitHub Pages**
   - `index.html` renderiza la UI.
   - `app.js` llama a la Edge Function.
   - Auto-refresh cada 30 segundos mientras hay sesión.

5. **Hermes/Walter worker**
   - Corre como cron cada ~2 minutos.
   - Script principal local:
     - `/root/.hermes/profiles/walter/scripts/walter_campaign_autoworker.py`
   - Abre teléfonos GeeLark, dispara WhatsApp, captura screenshots y registra eventos.
   - No debe cerrar teléfonos al terminar cada batch; debe cerrarlos al completar toda la campaña, agotar cap, cancelar o cuando Toni lo pida.

6. **GeeLark cloud phones**
   - Grupo lógico: `roma_toni`.
   - Perfiles usados: `Prueba T 01` a `Prueba T 10` y otros disponibles si están habilitados.
   - Cada envío requiere screenshot antes/después.

## Flujo de campaña live

1. Toni crea/selecciona campaña en dashboard.
2. Se configura:
   - `dry_run=false` para live.
   - `global_pause_min_seconds`.
   - `global_pause_max_seconds`.
   - `max_sends_per_device_per_day`.
   - grupo activo (`roma_toni`).
3. Dashboard crea un `campaign_run` live.
4. Worker toma el run activo.
5. Worker selecciona filas pendientes mayores al frontier monotónico.
6. En cada batch:
   - Usa todos los celulares disponibles/no restringidos/no cappeados.
   - Abre WhatsApp por deep-link.
   - Verifica pre-send por screenshot.
   - Envía solo si el chat está correcto.
   - Verifica post-send por screenshot.
   - Marca Sheet C como `enviado` solo después de verificar.
   - Registra evento y actualiza matriz.
7. Después del batch:
   - Espera una pausa global random entre min/max.
   - No cierra celulares entre batches.
8. Termina cuando:
   - Se agota la cola.
   - Todos los celulares llegan a cap.
   - Todos los disponibles quedan restringidos/banneados.
   - Run se pausa/cancela.
9. Al terminar toda la campaña:
   - Cierra teléfonos GeeLark usados.
   - Verifica status cerrado.
   - Deja logs finales.

## Estados principales de matriz

- `queued`: pendiente de abrir.
- `opening`: solicitud de apertura enviada.
- `opened`: teléfono abierto.
- `whatsapp_ready`: WhatsApp listo para preparar envío.
- `presend_check`: revisando screenshot antes de enviar.
- `sent`: envío verificado y Sheet marcado.
- `invalid`: destino no existe en WhatsApp; Sheet queda sin marcar.
- `restricted`: perfil emisor restringido; no se vuelve a usar.
- `failed`: fallo operativo no clasificado.

## Reglas críticas

- Sheet C es fuente de verdad para enviado.
- No marcar `enviado` sin screenshot post-send limpio.
- No avanzar hacia filas menores; frontier siempre es monotónico.
- Si una fila es inválida, se deja sin marcar y se retry-forward a una fila mayor.
- Si un emisor está restringido, se deshabilita para siguientes tandas.
- El dashboard puede mostrar logs y matriz por caminos distintos; si logs avanzan pero matriz no, backfill de `campaign_run_device_batches` desde eventos.
- No cerrar teléfonos entre batches.
- Dashboard refresca cada 30s.

## Deploy

El repo puede publicarse por GitHub Pages. La API real vive en Supabase Edge Functions.

Pasos mínimos:

1. Subir `index.html` y `app.js` a `main`.
2. Activar GitHub Pages desde `main` / root.
3. Configurar secrets reales fuera del repo.
4. Verificar en navegador que la UI renderiza, no HTML crudo.
5. Entrar con Basic Auth configurado en Supabase.

## Archivos

- `index.html`: UI del dashboard.
- `app.js`: lógica frontend/API polling.
- `.env.example`: nombres de variables necesarias sin valores reales.
- `docs/OPERACION.md`: runbook operativo completo.
- `docs/SECRETS.md`: cómo cargar credenciales de forma segura.

## Nota de privacidad

Repo privado no significa repo seguro para secretos. Cualquier credencial real debe ir como secret/env y rotarse si alguna vez se filtra.
