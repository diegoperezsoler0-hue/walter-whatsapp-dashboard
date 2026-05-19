# Operación punta a punta — Walter WhatsApp Live Campaign

## 1. Componentes

### Dashboard

- Host: GitHub Pages / estático.
- Archivos: `index.html`, `app.js`.
- Función: mostrar analytics, live runs, logs, parámetros, campañas y celulares.
- Refresh automático: cada 30 segundos con `setInterval(()=>auth&&loadState().catch(()=>{}),30000)`.

### API

- Supabase Edge Function `walter-dashboard`.
- El frontend llama rutas como:
  - `GET /api/state`
  - `POST /api/run/control`
  - `POST /api/agent/event`
  - endpoints de campañas, settings y devices.
- Protegida por Basic Auth.

### Base de datos

Tablas relevantes:

- `app_settings`: parámetros globales.
- `campaigns`: campañas configuradas.
- `campaign_runs`: cada ejecución live/dry-run.
- `campaign_run_events`: timeline auditable.
- `campaign_run_device_batches`: matriz batch×celular.
- `devices`: celulares/perfiles GeeLark.

### Worker Walter

- Cron Hermes: `Walter WhatsApp campaign autonomous worker`.
- Frecuencia: ~cada 2 minutos.
- Script local: `/root/.hermes/profiles/walter/scripts/walter_campaign_autoworker.py`.
- Responsabilidad: mover el run, abrir teléfonos, preparar WhatsApp, enviar, verificar, marcar Sheet y auditar.

### Fuentes externas

- Google Sheets: cola de destinatarios/mensajes/status.
- GeeLark: cloud phones Android.
- WhatsApp: app dentro del cloud phone.

## 2. Parámetros operativos

Parámetros clave esperados en `app_settings` o `campaign.config`:

- `active_group_key`: grupo de celulares, normalmente `roma_toni`.
- `active_campaign_id`: campaña activa.
- `dry_run`: `false` para campaña live real.
- `autonomy_mode`: `auto_worker`.
- `global_pause_min_seconds`: pausa mínima global entre batches.
- `global_pause_max_seconds`: pausa máxima global entre batches.
- `max_sends_per_device_per_day`: cap por celular.
- `single_send_test`: debe ser `false` para campaña completa.

## 3. Ciclo normal del worker

1. Leer `/api/state`.
2. Resolver campaña activa desde `campaign_runs.campaign_id` y `campaigns[]`.
3. Si `dry_run=true`, no abrir teléfonos ni enviar.
4. Si run está `paused`, no avanzar.
5. Si `next_batch_due_epoch` está en el futuro, esperar/salir silencioso.
6. Calcular frontier:
   - `campaign_runs.data.frontier_after` si existe.
   - si no, máximo `message_sent_verified.data.sheet_row`.
7. Leer Google Sheet desde la fila siguiente al frontier.
8. Seleccionar celulares disponibles:
   - enabled/use_for_sending.
   - no `restricted`/`banned`.
   - debajo de `max_sends_per_device_per_day`.
9. Crear/actualizar matriz del batch.
10. Abrir teléfonos GeeLark si no están corriendo.
11. Abrir WhatsApp con deep link para cada número/mensaje.
12. Capturar screenshot pre-send.
13. Clasificar:
    - chat válido con draft y send visible → enviar.
    - número inválido → registrar `invalid`, Sheet queda `FALSE`.
    - cuenta restringida → registrar `restricted`, celular fuera de uso.
    - modal temporal → resolver y re-screenshot.
    - `Buscando...` → esperar/reintentar una vez.
14. Tocar send solo si screenshot confirma estado correcto.
15. Capturar screenshot post-send.
16. Confirmar burbuja saliente y campo limpio.
17. Marcar Sheet C `enviado` y re-leer la celda.
18. Registrar `message_sent_verified` y matriz `sent`.
19. Retry-forward para celulares que tuvieron fila inválida: usar siguiente fila mayor.
20. Setear pausa global random entre min/max.
21. Mantener celulares abiertos para el próximo batch.
22. Cerrar celulares solo al terminar toda la campaña/cap/cancelación.

## 4. Frontier monotónico

La campaña nunca vuelve a filas menores.

Ejemplo:

- Frontier = 101.
- Batch 4 prueba filas 102–107.
- Si 102 falla por inválida y 108 se envía por retry-forward, la frontier avanza a 108.
- 102 queda `FALSE`, pero no vuelve a bloquear.

## 5. Cierre de celulares

No cerrar teléfonos después de cada batch.

Cerrar solo cuando:

- campaña completa;
- todos los teléfonos disponibles llegaron al cap;
- todos quedaron restringidos/banneados;
- Toni cancela/elimina run;
- se pide shutdown final.

Al cerrar:

1. Llamar GeeLark stop/close.
2. Poll de `/phone/status`.
3. Verificar status cerrado.
4. Registrar evento final.

## 6. Dashboard: logs vs matriz

Los logs vienen de `campaign_run_events`.
La matriz viene de `campaign_run_device_batches`.

Si los logs muestran Batch 3/4 pero la matriz no:

1. Consultar últimos eventos por `run_id`.
2. Agrupar último evento por `(batch_no, device_id)`.
3. Upsert en `campaign_run_device_batches` con `cell_status` derivado.
4. Actualizar `campaign_runs.current_batch` y `total_batches`.
5. Releer `/api/state`.

## 7. Verificación requerida

Para decir que una tanda avanzó:

- Logs del batch visibles.
- Matriz del batch visible.
- Screenshots post-send verificados.
- Sheet C re-leído como `enviado` para filas enviadas.
- Filas inválidas/restringidas quedan sin marcar.
- Frontier registrado.
- Próxima pausa global registrada.

## 8. Errores frecuentes

- Cerrar teléfonos entre batches: incorrecto.
- Dejar dashboard sin refresh visible: incorrecto.
- Marcar Sheet sin screenshot: incorrecto.
- Tocar send desde pantalla incorrecta: incorrecto.
- Logs avanzan pero matriz no: bug de sync, reparar matriz.
- Usar `single_send_test=true` en campaña completa: incorrecto.
- Subir secretos al repo: prohibido.
