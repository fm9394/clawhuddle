# Anthropic Multi-Key Fallback Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Allow admins to configure multiple Anthropic API keys with priority ordering; OpenClaw handles automatic fallback on rate limits.

**Architecture:** Add `priority` column to `api_keys` table, remove single-key-per-provider constraint for Anthropic, generate `auth-profiles.json` with `order` field. Frontend shows key list with up/down arrow reordering.

**Tech Stack:** SQLite migration, Fastify API, React (Next.js), `@clawhuddle/shared` types

---

### Task 1: DB Migration — Add `priority` Column

**Files:**
- Create: `apps/api/src/db/migrations/011-api-key-priority.sql`

**Step 1: Create migration file**

```sql
ALTER TABLE api_keys ADD COLUMN priority INTEGER NOT NULL DEFAULT 0;
```

**Step 2: Run migration to verify**

Run: `cd apps/api && npx tsx src/db/migrate.ts`
Expected: `Migration applied: 011-api-key-priority.sql`

**Step 3: Commit**

```bash
git add apps/api/src/db/migrations/011-api-key-priority.sql
git commit -m "feat: add priority column to api_keys table"
```

---

### Task 2: Shared Types — Add `priority` to API Types

**Files:**
- Modify: `packages/shared/src/index.ts` (around line 167, `SetApiKeyRequest`)

**Step 1: Update `SetApiKeyRequest` — no changes needed** (priority is server-assigned)

No type changes needed for the request. But we need to add `priority` to the display type. Currently `ApiKeyDisplay` lives in the frontend component (`apps/web/components/admin/api-key-form.tsx:9-16`). Move it to shared or just add `priority` to the API response — the frontend component defines its own `ApiKeyDisplay` interface, so we just need to add `priority` there.

**Step 2: Commit** (skip if nothing changed — proceed to Task 3)

---

### Task 3: API — Allow Multiple Keys Per Provider + Reorder Endpoint

**Files:**
- Modify: `apps/api/src/routes/org/api-keys.ts`

**Step 1: Update POST endpoint to append instead of replace**

In `apps/api/src/routes/org/api-keys.ts`, replace the POST handler body (lines 51-74).

Remove this line:
```typescript
db.prepare('DELETE FROM api_keys WHERE provider = ? AND is_company_default = 1 AND org_id = ?').run(provider, request.orgId!);
```

Replace with logic to compute next priority:
```typescript
const maxRow = db.prepare(
  'SELECT MAX(priority) as max_p FROM api_keys WHERE provider = ? AND is_company_default = 1 AND org_id = ?'
).get(provider, request.orgId!) as { max_p: number | null } | undefined;
const nextPriority = (maxRow?.max_p ?? -1) + 1;
```

Update the INSERT to include priority:
```typescript
db.prepare(
  'INSERT INTO api_keys (id, provider, key_value, is_company_default, org_id, credential_type, default_model, priority) VALUES (?, ?, ?, 1, ?, ?, ?, ?)'
).run(id, provider, encodeKey(key), request.orgId!, ct, defaultModel || null, nextPriority);
```

**Step 2: Update GET endpoint to return `priority`**

In the GET handler (line 34-43), add `priority` to the returned data:
```typescript
return {
  data: keys.map((k) => ({
    ...k,
    key_masked: maskKey(decodeKey(k.key_value)),
    key_value: undefined,
    credential_type: k.credential_type || 'api_key',
    default_model: k.default_model || null,
    priority: k.priority ?? 0,
  })),
};
```

Change the ORDER BY to `ORDER BY priority ASC, created_at DESC`.

**Step 3: Add PUT `/api-keys/reorder` endpoint**

Add after the DELETE handler:
```typescript
// Reorder API keys for a provider
app.put<{ Body: { provider: string; keyIds: string[] } }>(
  '/api/orgs/:orgId/api-keys/reorder',
  { preHandler: requireRole('owner', 'admin') },
  async (request, reply) => {
    const { provider, keyIds } = request.body;
    if (!provider || !keyIds?.length) {
      return reply.status(400).send({ error: 'validation', message: 'provider and keyIds are required' });
    }
    const db = getDb();
    const updateStmt = db.prepare('UPDATE api_keys SET priority = ? WHERE id = ? AND org_id = ?');
    const txn = db.transaction(() => {
      for (let i = 0; i < keyIds.length; i++) {
        updateStmt.run(i, keyIds[i], request.orgId!);
      }
    });
    txn();
    syncAuthProfiles(request.orgId!);
    return { data: { provider, order: keyIds } };
  }
);
```

**Step 4: Update `getOrgAllApiKeys()` to order by priority**

Change the SQL in `getOrgAllApiKeys()` (line 122-124):
```typescript
const rows = db.prepare(
  'SELECT provider, key_value, credential_type, default_model FROM api_keys WHERE is_company_default = 1 AND org_id = ? ORDER BY priority ASC'
).all(orgId) as ...;
```

**Step 5: Commit**

```bash
git add apps/api/src/routes/org/api-keys.ts
git commit -m "feat: allow multiple API keys per provider with priority ordering"
```

---

### Task 4: Gateway — Generate Multi-Profile auth-profiles.json with `order`

**Files:**
- Modify: `apps/api/src/services/gateway.ts` (function `writeAuthProfiles`, lines 174-240)

**Step 1: Update `writeAuthProfiles()` to handle multiple keys per provider**

The current code uses profile IDs like `${provider}:setup-token` or `${provider}:manual`. When there are multiple keys for the same provider, we need unique IDs and an `order` map.

Replace the profile ID generation logic. Track per-provider counters:

```typescript
function writeAuthProfiles(orgId: string, userId: string): { providerIds: string[]; modelOverrides: Record<string, string> } {
  const allKeys = getOrgAllApiKeys(orgId);
  const profiles: Record<string, Record<string, unknown>> = {};
  const providerIds: string[] = [];
  const modelOverrides: Record<string, string> = {};
  const order: Record<string, string[]> = {};
  const providerCounters: Record<string, number> = {};

  for (const { provider, key, credential_type, default_model } of allKeys) {
    const providerConfig = PROVIDERS.find((p) => p.id === provider);
    if (!providerConfig) continue;
    if (!providerIds.includes(provider)) providerIds.push(provider);
    if (default_model && !modelOverrides[provider]) modelOverrides[provider] = default_model;

    // Generate unique profile ID
    const count = (providerCounters[provider] ?? 0) + 1;
    providerCounters[provider] = count;
    const suffix = count === 1 ? '' : `-${count}`;

    let profileId: string;

    if (credential_type === "oauth") {
      try {
        const oauth = JSON.parse(key);
        const tokens = oauth.tokens ?? oauth;
        if (!tokens.access_token || !tokens.refresh_token) continue;

        let expires: number | undefined;
        try {
          const payload = JSON.parse(
            Buffer.from(tokens.access_token.split(".")[1], "base64").toString(),
          );
          if (payload.exp) expires = payload.exp;
        } catch { /* non-JWT */ }

        profileId = `${provider}:oauth${suffix}`;
        profiles[profileId] = {
          type: "oauth",
          provider,
          access: tokens.access_token,
          refresh: tokens.refresh_token,
          ...(expires ? { expires } : {}),
        };
      } catch {
        continue;
      }
    } else if (credential_type === "token") {
      profileId = `${provider}:setup-token${suffix}`;
      profiles[profileId] = {
        type: "token",
        provider,
        token: key,
      };
    } else {
      profileId = `${provider}:manual${suffix}`;
      profiles[profileId] = { type: "api_key", provider, key };
    }

    if (!order[provider]) order[provider] = [];
    order[provider].push(profileId);
  }

  // Only include order for providers with multiple profiles
  const orderFiltered: Record<string, string[]> = {};
  for (const [provider, ids] of Object.entries(order)) {
    if (ids.length > 1) orderFiltered[provider] = ids;
  }

  const authProfilesPath = path.join(
    getGatewayDir(orgId, userId),
    "agents", "main", "agent", "auth-profiles.json",
  );
  fs.mkdirSync(path.dirname(authProfilesPath), { recursive: true });
  fs.writeFileSync(
    authProfilesPath,
    JSON.stringify({
      version: 1,
      profiles,
      ...(Object.keys(orderFiltered).length > 0 ? { order: orderFiltered } : {}),
    }, null, 2),
  );

  return { providerIds, modelOverrides };
}
```

**Step 2: Commit**

```bash
git add apps/api/src/services/gateway.ts
git commit -m "feat: generate multi-profile auth-profiles.json with order field"
```

---

### Task 5: Frontend — Multi-Key UI for Anthropic with Reordering

**Files:**
- Modify: `apps/web/components/admin/api-key-form.tsx`

**Step 1: Update `ApiKeyDisplay` to include `priority`**

```typescript
export interface ApiKeyDisplay {
  id: string;
  provider: string;
  key_masked: string;
  is_company_default: boolean;
  credential_type?: CredentialType;
  default_model?: string | null;
  priority?: number;
}
```

**Step 2: Change `currentKey` to `currentKeys` — return array for Anthropic**

Replace `const currentKey = (provider: string) => keys.find(...)` with:
```typescript
const keysForProvider = (provider: string) =>
  keys.filter((k) => k.provider === provider).sort((a, b) => (a.priority ?? 0) - (b.priority ?? 0));
```

**Step 3: Add reorder function**

```typescript
const reorderKeys = async (provider: string, keyIds: string[]) => {
  try {
    await fetchFn('/api-keys/reorder', {
      method: 'PUT',
      body: JSON.stringify({ provider, keyIds }),
    });
    await refresh();
    toast('Priority updated', 'success');
  } catch (err: any) {
    toast(err.message, 'error');
  }
};

const moveKey = (provider: string, existingKeys: ApiKeyDisplay[], index: number, direction: 'up' | 'down') => {
  const newKeys = [...existingKeys];
  const swapIdx = direction === 'up' ? index - 1 : index + 1;
  if (swapIdx < 0 || swapIdx >= newKeys.length) return;
  [newKeys[index], newKeys[swapIdx]] = [newKeys[swapIdx], newKeys[index]];
  reorderKeys(provider, newKeys.map((k) => k.id));
};
```

**Step 4: Update render — show key list for Anthropic**

Replace the single `existing` key display block with a list. For Anthropic (`id === 'anthropic'`), render all keys. For other providers, keep single-key behavior.

The existing keys section (currently lines 185-215) becomes:

```tsx
{(() => {
  const providerKeys = keysForProvider(id);
  // Multi-key provider (Anthropic): show list
  if (id === 'anthropic' && providerKeys.length > 0) {
    return (
      <div className="space-y-1 mb-3">
        {providerKeys.map((k, idx) => (
          <div key={k.id} className="flex items-center gap-2">
            <span className="text-[10px] text-right w-4" style={{ color: 'var(--text-tertiary)' }}>
              {idx + 1}.
            </span>
            <span
              className="text-[10px] font-medium px-1.5 py-0.5 rounded"
              style={{
                background: k.credential_type === 'api_key' ? 'var(--bg-tertiary)' : 'var(--accent-muted, rgba(99,102,241,0.15))',
                color: k.credential_type === 'api_key' ? 'var(--text-tertiary)' : 'var(--accent)',
              }}
            >
              {CRED_TYPE_LABEL[k.credential_type ?? 'api_key']}
            </span>
            <code
              className="px-1.5 py-0.5 rounded text-[11px] font-mono"
              style={{ background: 'var(--bg-tertiary)', color: 'var(--text-secondary)' }}
            >
              {k.key_masked}
            </code>
            <div className="flex gap-0.5 ml-auto">
              <button
                onClick={() => moveKey(id, providerKeys, idx, 'up')}
                disabled={idx === 0 || saving}
                className="text-xs px-1 py-0.5 rounded disabled:opacity-30"
                style={{ color: 'var(--text-tertiary)' }}
                title="Move up (higher priority)"
              >
                ↑
              </button>
              <button
                onClick={() => moveKey(id, providerKeys, idx, 'down')}
                disabled={idx === providerKeys.length - 1 || saving}
                className="text-xs px-1 py-0.5 rounded disabled:opacity-30"
                style={{ color: 'var(--text-tertiary)' }}
                title="Move down (lower priority)"
              >
                ↓
              </button>
              <button
                onClick={() => deleteKey(k.id, label)}
                disabled={saving}
                className="text-xs px-2 py-0.5 rounded transition-colors disabled:opacity-50"
                style={{ color: 'var(--text-tertiary)' }}
                onMouseEnter={(e) => { e.currentTarget.style.color = 'var(--error, #ef4444)'; }}
                onMouseLeave={(e) => { e.currentTarget.style.color = 'var(--text-tertiary)'; }}
              >
                Delete
              </button>
            </div>
          </div>
        ))}
      </div>
    );
  }
  // Single-key providers: existing behavior
  const existing = providerKeys[0];
  if (!existing) return null;
  return (
    <div className="flex items-center gap-2 mb-3">
      <span
        className="text-[10px] font-medium px-1.5 py-0.5 rounded"
        style={{
          background: existing.credential_type === 'api_key' ? 'var(--bg-tertiary)' : 'var(--accent-muted, rgba(99,102,241,0.15))',
          color: existing.credential_type === 'api_key' ? 'var(--text-tertiary)' : 'var(--accent)',
        }}
      >
        {CRED_TYPE_LABEL[existing.credential_type ?? 'api_key']}
      </span>
      <p className="text-xs" style={{ color: 'var(--text-tertiary)' }}>
        <code
          className="px-1.5 py-0.5 rounded text-[11px] font-mono"
          style={{ background: 'var(--bg-tertiary)', color: 'var(--text-secondary)' }}
        >
          {existing.key_masked}
        </code>
      </p>
      <button
        onClick={() => deleteKey(existing.id, label)}
        disabled={saving}
        className="text-xs px-2 py-0.5 rounded transition-colors disabled:opacity-50"
        style={{ color: 'var(--text-tertiary)' }}
        onMouseEnter={(e) => { e.currentTarget.style.color = 'var(--error, #ef4444)'; }}
        onMouseLeave={(e) => { e.currentTarget.style.color = 'var(--text-tertiary)'; }}
      >
        Delete
      </button>
    </div>
  );
})()}
```

**Step 5: Update model selector to use first key**

The model selector `onChange` currently uses `existing` — update to use `providerKeys[0]`:
```typescript
const firstKey = keysForProvider(id)[0];
// In onChange:
if (firstKey) {
  updateModel(firstKey.id, id, val);
}
```

**Step 6: Commit**

```bash
git add apps/web/components/admin/api-key-form.tsx
git commit -m "feat: multi-key UI for Anthropic with priority reordering"
```

---

### Task 6: Integration Test — Verify End-to-End

**Step 1: Run the dev server**

```bash
cd apps/api && npx tsx src/db/migrate.ts
npm run dev
```

**Step 2: Manual verification checklist**

- [ ] Add first Anthropic key → appears as #1
- [ ] Add second Anthropic key → appears as #2 below first
- [ ] Click ↑ on second key → becomes #1, first becomes #2
- [ ] Click ↓ on first key → swaps back
- [ ] Delete one key → list updates correctly
- [ ] Other providers (OpenAI, etc.) still show single-key behavior
- [ ] Check `data/gateways/<orgId>/<userId>/agents/main/agent/auth-profiles.json` has `order` field with both profiles

**Step 3: Final commit**

```bash
git add -A
git commit -m "feat: Anthropic multi-key fallback with priority ordering"
```
