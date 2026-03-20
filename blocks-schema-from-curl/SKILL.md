---
name: blocks-schema-from-curl
description: Create and publish schemas in the SELISE Blocks DataGateway using a browser cURL. Use when you need to add a new schema (with fields and access control) without clicking through the Blocks Cloud UI. The user extracts one authenticated cURL from their browser and hands it to the agent, which handles the rest via the UDS REST API.
argument-hint: "[schema-name] [fields]"
disable-model-invocation: true
---

You are helping the user create and publish a schema in the SELISE Blocks DataGateway. You need one authenticated cURL from the user's browser — that gives you the session cookie and project key. Everything else is done by you via the API.

Follow the steps below in order.

---

## Step 1 — Ask the user for a cURL from Blocks Cloud

Tell the user:

> I need one authenticated request from your browser to talk to the Blocks API. Here's how to grab it in 30 seconds:
>
> 1. Open **[https://cloud.seliseblocks.com](https://cloud.seliseblocks.com)** and make sure you are logged in
> 2. Open **DevTools** → **Network** tab (`F12` or `Cmd+Option+I`, then click Network)
> 3. Reload the page or click anything — any request to `api.seliseblocks.com` will do
> 4. Find any request in the list that goes to `api.seliseblocks.com` (filter by XHR/Fetch if needed)
> 5. Right-click that request → **Copy** → **Copy as cURL**
> 6. Paste it here

You only need this once per session. The cookie inside it is your auth token.

---

## Step 2 — Extract credentials from the cURL

From the pasted cURL, extract:

- **Session cookie**: the `Cookie:` header value (contains `access_token_...=eyJ...`)
- **Admin x-blocks-key**: the `x-blocks-key` header value (this is the **tenant key**, different from the project key)
- **Project key**: the `ProjectKey=` query parameter if present, OR ask the user for it (also found in `.env` as `VITE_X_BLOCKS_KEY`)

Store these as variables for all subsequent API calls:

```bash
AUTH_COOKIE='access_token_...=eyJ...'   # from Cookie header
ADMIN_KEY='...'                          # from x-blocks-key header
PROJECT_KEY='...'                        # from ProjectKey= param or .env
```

> **Note:** The `x-blocks-key` in the admin cURL is the **tenant-level** key (used for schema management). The `VITE_X_BLOCKS_KEY` / `ProjectKey` is the **project-level** key (used for DataGateway queries). Both are needed.

---

## Step 3 — Design the schema with the user

Ask the user what entity they want to store. For each field, determine:

- **Field name**: PascalCase (e.g. `PresetName`, `SettingsJson`, `UserId`)
- **Field type**: `String`, `Int`, `Float`, `Boolean`, or `DateTime`

Also agree on access control for each operation:

| Level | Value | Meaning |
|-------|-------|---------|
| Public | `2` | Anyone can perform this operation |
| Logged in users | `1` | Only authenticated project users |
| No access | `0` | Blocked |

Typical patterns:
- **Public read + write, restricted edit/delete**: read=2, write=2, edit=1, delete=1
- **Fully public** (prototyping): read=2, write=2, edit=2, delete=2
- **Auth required for everything**: read=1, write=1, edit=1, delete=1

Confirm the schema name, fields, and access levels with the user before proceeding.

---

## Step 4 — Create the schema via API

Use `POST /uds/v1/schemas/define` with the credentials extracted in Step 2.

```bash
curl -s -X POST 'https://api.seliseblocks.com/uds/v1/schemas/define' \
  -H 'accept: application/json' \
  -H 'content-type: application/json' \
  -H "x-blocks-key: $ADMIN_KEY" \
  -H 'origin: https://cloud.seliseblocks.com' \
  -b "$AUTH_COOKIE" \
  -d '{
    "schemaName": "YourSchemaName",
    "collectionName": "YourSchemaNames",
    "projectKey": "'"$PROJECT_KEY"'",
    "schemaType": 1,
    "readAccessLevel": 2,
    "writeAccessLevel": 2,
    "editAccessLevel": 1,
    "deleteAccessLevel": 1,
    "fields": [
      { "name": "FieldOne", "type": "String",  "isArray": false, "fields": [] },
      { "name": "FieldTwo", "type": "Int",     "isArray": false, "fields": [] },
      { "name": "FieldThree", "type": "Boolean", "isArray": false, "fields": [] }
    ]
  }'
```

**Rules:**
- `schemaName`: PascalCase, no spaces → `HonkGamePreset`
- `collectionName`: same + trailing `s` → `HonkGamePresets`
- `schemaType`: always `1` for Entity
- Access levels: use the values agreed in Step 3

A successful response looks like:
```json
{
  "isSuccess": true,
  "data": { "acknowledged": true, "itemId": "uuid-here" }
}
```

If `isSuccess` is false, show the user the error message and stop.

---

## Step 5 — Publish the schema

Schema changes are staged until reloaded. Use `POST /uds/v1/configurations/{projectKey}/reload`:

```bash
curl -s -X POST "https://api.seliseblocks.com/uds/v1/configurations/$PROJECT_KEY/reload" \
  -H 'accept: application/json' \
  -H 'content-type: application/json' \
  -H "x-blocks-key: $ADMIN_KEY" \
  -H 'origin: https://cloud.seliseblocks.com' \
  -b "$AUTH_COOKIE" \
  -d '{}'
```

A successful response:
```json
{ "isSuccess": true, "message": "Schema reloaded successfully." }
```

The DataGateway now exposes the new schema as GraphQL operations.

---

## Step 6 — Verify

Confirm the schema is live by querying the DataGateway:

```bash
curl -s -X POST "https://api.seliseblocks.com/uds/v1/$PROJECT_SLUG/gateway" \
  -H 'content-type: application/json' \
  -H "x-blocks-key: $PROJECT_KEY" \
  --data-raw '{"query":"query { getYourSchemaNames(input:{pageNo:1,pageSize:1}){ totalCount } }"}'
```

Replace `$PROJECT_SLUG` with the short slug from the hosted URL (e.g. `dsjogn` from `https://dsjogn-dqnxc.seliseblocks.com`).

Expected response:
```json
{ "data": { "getYourSchemaNames": { "totalCount": 0 } } }
```

A zero count with no errors means the schema is live and ready.

---

## Step 7 — Update an existing schema (optional)

To add fields to an existing schema, first get its `id`:

```bash
curl -s "https://api.seliseblocks.com/uds/v1/schemas?Keyword=YourSchemaName&PageSize=1&PageNo=1&ProjectKey=$PROJECT_KEY" \
  -H "x-blocks-key: $ADMIN_KEY" \
  -b "$AUTH_COOKIE" | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['data']['items'][0]['id'])"
```

Then update using `PUT /uds/v1/schemas/define`:

```bash
curl -s -X PUT 'https://api.seliseblocks.com/uds/v1/schemas/define' \
  -H 'content-type: application/json' \
  -H "x-blocks-key: $ADMIN_KEY" \
  -b "$AUTH_COOKIE" \
  -d '{
    "itemId": "uuid-from-above",
    "schemaName": "YourSchemaName",
    "collectionName": "YourSchemaNames",
    "schemaType": 1,
    "readAccessLevel": 2,
    "writeAccessLevel": 2,
    "editAccessLevel": 1,
    "deleteAccessLevel": 1,
    "fields": [
      { "name": "ExistingField", "type": "String", "isArray": false, "fields": [] },
      { "name": "NewField",      "type": "Int",    "isArray": false, "fields": [] }
    ]
  }'
```

Always re-run the reload call from Step 5 after any schema change.

---

## Common errors

| Error | Cause | Fix |
|-------|-------|-----|
| `401 Unauthorized` | Cookie expired | Ask the user to copy a fresh cURL from the browser |
| `403 Forbidden` | Wrong `x-blocks-key` (project key used instead of tenant key) | Use the key from the cURL's `x-blocks-key` header, not from `.env` |
| `400` + validation message | Field type typo or missing required property | Check field `type` is one of: `String`, `Int`, `Float`, `Boolean`, `DateTime` |
| Schema created but GraphQL returns unknown field | Reload not called after create/update | Run the reload call from Step 5 |
| `isSuccess: false` + `"already exists"` | Schema name already taken | Use a different name or use PUT to update the existing schema |
