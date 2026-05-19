# Credenciales y secretos

## Regla principal

No subir credenciales reales al repositorio, aunque sea privado.

Motivo: un repo privado puede compartirse, clonarse, filtrarse, quedar en backups o logs. Las credenciales reales deben vivir en entornos seguros y rotarse si se exponen.

## Qué NO versionar

- `GEELARK_BEARER_TOKEN`
- `SUPABASE_SERVICE_ROLE_KEY`
- `SUPABASE_DB_PASSWORD`
- Google OAuth refresh/access tokens
- Proxy usernames/passwords reales
- Dashboard Basic Auth password real
- GitHub tokens
- URLs de `/phone/start` de GeeLark con token
- Dumps de screenshots con datos sensibles, salvo evidencia operativa local controlada

## Dónde guardar cada secreto

### Supabase Edge Function

Configurar con Supabase secrets:

```bash
supabase secrets set WALTER_DASHBOARD_USER="..."
supabase secrets set WALTER_DASHBOARD_PASS="..."
supabase secrets set SUPABASE_SERVICE_ROLE_KEY="..."
supabase secrets set GEELARK_BEARER_TOKEN="..."
```

### Hermes/Walter host

Guardar en variables de entorno o archivos `.env` locales no versionados.

Ejemplo local:

```bash
export GEELARK_BEARER_TOKEN="..."
export SUPABASE_URL="..."
export SUPABASE_SERVICE_ROLE_KEY="..."
export GOOGLE_APPLICATION_CREDENTIALS="/ruta/local/no-versionada.json"
```

### GitHub Actions

Si hace falta CI/CD:

```bash
gh secret set SUPABASE_ACCESS_TOKEN --body "..."
gh secret set SUPABASE_PROJECT_REF --body "..."
gh secret set WALTER_DASHBOARD_USER --body "..."
gh secret set WALTER_DASHBOARD_PASS --body "..."
```

GitHub Secrets guardan valores cifrados y no los exponen en el repo.

## Archivos permitidos

- `.env.example`: nombres de variables sin valores reales.
- documentación con placeholders.
- instrucciones de rotación/configuración.

## Rotación

Si una credencial real se subió alguna vez:

1. Revocar/rotar inmediatamente en el proveedor.
2. Eliminar del repo.
3. Purgar historial si corresponde.
4. Reconfigurar secreto nuevo en Supabase/GitHub/Hermes.
5. Verificar que la app funcione con el nuevo secreto.

## Checklist antes de commit

Ejecutar búsqueda local:

```bash
grep -RInE "(Bearer|service_role|password|passwd|token|api[_-]?key|secret|refresh_token|proxy|supabase.*key)" . --exclude-dir=.git
```

Revisar manualmente cualquier hallazgo.
