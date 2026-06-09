# Supabase Studio JS Patches for Self-Hosted Mode

**Date:** 2026-05-31
**Studio Image:** `supabase/studio:2026.04.08-sha-205cbe7`
**Context:** Pigsty v4.3.0 self-hosted Supabase deployment on 89.167.105.66

---

## Summary

Self-hosted Supabase Studio is missing several Cloud-only API endpoints, causing features to silently fail. Three patches were applied to the Studio container's JS chunks to fix:

1. AI chat "Run" button on SQL blocks not working (missing database connection string)
2. Permission check blocking SQL snippet creation from AI chat
3. Guard conditions (profile/project checks) blocking snippet creation

**All patches are inside the container and will be lost on container recreation.**

---

## Patch 1: Database Connection String Fallback

### Problem
The AI assistant chat renders SQL code blocks with a "Run" button. Clicking it did nothing — no request, no error. The handler checks for a `connectionString` from the `usePrimaryDatabase` hook, which calls `/api/platform/database/{ref}`. This endpoint returns **404** in self-hosted mode, so `connectionString` is `undefined` and the handler silently exits.

### Root Cause
```js
// In the AI chat SQL block component:
{database: B} = usePrimaryDatabase({projectRef: v})
M = B?.connection_string_read_only
O = B?.connectionString  // undefined → no connection → no execution

// Later in the execute handler:
let s = "mutation" === t ? O : M ?? O
if (!s) return o?.({messageId: e, errorText: "Unable to find a database connection..."})
```

### Fix
**File:** `/app/apps/studio/.next/static/chunks/58d5b46be32ee3fa.js`

```bash
# Original:
M=B?.connection_string_read_only,O=B?.connectionString,

# Patched to add fallback:
M=B?.connection_string_read_only,O=B?.connectionString||"postgresql://supabase_admin:DBUser.Supa@89.167.105.66:5432/postgres",
```

### Apply
```bash
docker exec supabase-studio sed -i \
  's/M=B?.connection_string_read_only,O=B?.connectionString,/M=B?.connection_string_read_only,O=B?.connectionString||"postgresql:\/\/supabase_admin:DBUser.Supa@89.167.105.66:5432\/postgres",/' \
  /app/apps/studio/.next/static/chunks/58d5b46be32ee3fa.js
```

---

## Patch 2: Permission Check Bypass

### Problem
The `useAsyncCheckPermissions` hook calls `/api/platform/permissions` which returns **404** in self-hosted mode. The `can` result is `undefined`/`false`, causing the snippet creation guard to block with a toast: "Your queries will not be saved as you do not have sufficient permissions."

### Root Cause
```js
{can: _} = useAsyncCheckPermissions(PermissionAction.CREATE, "user_content", ...)
if(!_)return(0,o.toast)("Your queries will not be saved as you do not have sufficient permissions")
```

### Fix
Replace `if(!_)` with `if(!1)` (always false = never blocks) in ALL static chunks.

```bash
docker exec supabase-studio sh -c '
  for f in /app/apps/studio/.next/static/chunks/*.js; do
    sed -i "s/if(!_)return(0,o.toast)/if(!1)return(0,o.toast)/g" "$f"
  done
'
```

---

## Patch 3: Profile/Project Guard Bypass

### Problem
The snippet creation handler also checks for `projectRef`, `project`, and `profile` via `console.error` guards. These can silently fail if the async state hasn't resolved.

### Root Cause
```js
if(!s)return console.error("Project ref is required")
if(!v)return console.error("Project is required")
if(!r)return console.error("Profile is required")
```

### Fix
Replace all three guards with `if(!1)` in ALL static chunks.

```bash
docker exec supabase-studio sh -c '
  for f in /app/apps/studio/.next/static/chunks/*.js; do
    sed -i "s/if(!s)return console.error(\"Project ref is required\")/if(!1)return console.error(\"Project ref is required\")/g" "$f"
    sed -i "s/if(!v)return console.error(\"Project is required\")/if(!1)return console.error(\"Project is required\")/g" "$f"
    sed -i "s/if(!r)return console.error(\"Profile is required\")/if(!1)return console.error(\"Profile is required\")/g" "$f"
  done
'
```

---

## Database Fixes (Persistent)

These SQL changes were made directly to PostgreSQL and persist across container restarts.

### Missing `supabase_migrations` Schema

Studio expects `supabase_migrations.schema_migrations` to exist.

```sql
CREATE SCHEMA IF NOT EXISTS supabase_migrations;
CREATE TABLE IF NOT EXISTS supabase_migrations.schema_migrations (
    version text NOT NULL,
    statements text[],
    name text
);
ALTER TABLE supabase_migrations.schema_migrations OWNER TO supabase_admin;
```

### RLS on `oban_jobs`

```sql
ALTER TABLE public.oban_jobs ENABLE ROW LEVEL SECURITY;
REVOKE ALL ON public.oban_jobs FROM anon, authenticated;
GRANT ALL ON public.oban_jobs TO service_role;
```

---

## Re-apply All Patches After Container Recreate

```bash
#!/bin/bash
# Re-apply all Studio JS patches after docker compose up

CONTAINER="supabase-studio"
CONNSTR="postgresql://supabase_admin:DBUser.Supa@89.167.105.66:5432/postgres"

# Wait for container to be healthy
echo "Waiting for $CONTAINER to be healthy..."
until docker inspect --format='{{.State.Health.Status}}' $CONTAINER 2>/dev/null | grep -q healthy; do
  sleep 2
done

# Patch 1: Connection string fallback
docker exec $CONTAINER sed -i \
  "s/M=B?.connection_string_read_only,O=B?.connectionString,/M=B?.connection_string_read_only,O=B?.connectionString||\"${CONNSTR}\",/" \
  /app/apps/studio/.next/static/chunks/58d5b46be32ee3fa.js

# Patch 2: Permission check bypass
docker exec $CONTAINER sh -c '
  for f in /app/apps/studio/.next/static/chunks/*.js; do
    sed -i "s/if(!_)return(0,o.toast)/if(!1)return(0,o.toast)/g" "$f"
  done
'

# Patch 3: Profile/project guard bypass
docker exec $CONTAINER sh -c '
  for f in /app/apps/studio/.next/static/chunks/*.js; do
    sed -i "s/if(!s)return console.error(\"Project ref is required\")/if(!1)return console.error(\"Project ref is required\")/g" "$f"
    sed -i "s/if(!v)return console.error(\"Project is required\")/if(!1)return console.error(\"Project is required\")/g" "$f"
    sed -i "s/if(!r)return console.error(\"Profile is required\")/if(!1)return console.error(\"Profile is required\")/g" "$f"
  done
'

echo "All patches applied. Hard refresh browser (Ctrl+Shift+R)."
```

---

## Missing Self-Hosted API Endpoints

These Studio endpoints return 404 in self-hosted mode and are the underlying cause:

| Endpoint | Purpose | Impact |
|----------|---------|--------|
| `/api/platform/database/{ref}` | Database connection info | AI chat can't execute SQL |
| `/api/platform/permissions` | Permission checks | Snippet creation blocked |
| `/api/platform/notifications` | User notifications | Harmless 404 |
