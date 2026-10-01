---
name: manage-data-view
description: Author, publish, inspect and execute Pivotly data views (governed, versioned, read-only parameterized SQL queries managed as config items) in direct_sql mode through the Pivotly MCP tool config-item_data-view, with careful handling of ${p_...} parameter references. Use it whenever someone mentions a Pivotly data view, a data_view config item, or a parameterized query over usdf tables, or says things like 'create a data view', 'write a view that returns active clients by region', 'add a parameter to the view', 'make region optional', 'my parameter isn't being substituted', 'do I need ${} around the parameter', 'the view ignores p_region', 'INVALID_PARAMETER_NAME_PREFIX', 'publish the data view', 'run the view for APAC', 'execute active-clients', 'what parameters does this view take', 'show me the view definition', 'which views are published', 'publication history', 'unpublish the view', 'delete the data view', 'my view says publish it first', 'rejected for usdf clients_b' - even when Pivotly isn't named.
category: core
---

```skill
category: core
```

# Manage Pivotly Data Views (direct SQL)

## Safety / confirmation policy

Applies to every call in this skill, not only inside the workflows.

| Operation(s)                                                                                                        | Class                                  | Rule                                                                                                                                                                       |
| ------------------------------------------------------------------------------------------------------------------- | -------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `get`, `get_by_slug`, `list`, `filter`, `list_published`, `publications_for_view`, `get_definition`; `pivotly_ping` | read-only                              | Call freely; answer questions from these before touching anything.                                                                                                         |
| `execute`                                                                                                           | runs a read-only query                 | No confirmation needed, but confirm the parameter values when you had to infer any of them. Never execute a view to "test" it with values the user didn't give or approve. |
| `create`, `update`                                                                                                  | ⚠️ writes configuration                | Show the exact `slug`, `direct_sql` and `parameters[]` you will send and get an explicit yes first. Saving does not change the runnable view.                              |
| `publish`                                                                                                           | ⚠️ builds / replaces the runnable view | Confirm the slug and that the latest **saved** manifest is what they want live.                                                                                            |
| `unpublish`                                                                                                         | ⚠️ drops the runnable view             | Confirm the slug; after this, `execute` returns 404 until re-published. The config item remains.                                                                           |
| `delete_config_item`                                                                                                | ⚠️ soft-deletes the config item        | Confirm slug and id. Does **not** drop a published view function — that is `unpublish`. When the user says "delete the view", ask which they mean.                         |

Never guess a slug, id, or parameter name — look it up (`get_by_slug`, `list`, `get_definition`). Never retry a mutating call on a 5xx without reading `message` first (ordinary rejections arrive as 5xx — see Pitfalls).

**Never splice user-supplied values into `direct_sql`.** Every value that varies per run (a region, a date, an id) becomes a declared parameter referenced as `${p_<name>}` and is passed at `execute`. `direct_sql` holds only fixed query text.

## Mental model

A **data view** is a governed, versioned, **read-only parameterized query** over published data. You author it as a **config item** (`item_type` = `data_view`, identified by `slug` and a uuid `id`) whose `cfg_data` manifest carries the SQL (`direct_sql`) and the typed `parameters[]`.

Lifecycle: **create → (update) → publish → execute**, then **unpublish** or soft-delete when retired. Create/update only store the manifest; **publish is where the query and parameters are validated** and compiled into a runnable view function. Only a view with a `status=complete`, `implementation=function` publication can be executed.

**Parameter references:** a parameter declared as `{ "name": "p_region" }` is referenced in `direct_sql` as `${p_region}` and is passed to `execute` as `"parameters": { "p_region": "APAC" }`. The same exact name — `p_` prefix included — is used in all three places (wrapped in `${…}` only inside the SQL). Publish does not add the prefix for you.

Identity by operation: config-item `id` (uuid) → `update`, `delete_config_item`, `get`; `slug` → everything else (`create`, `update`, `get_by_slug`, `publish`, `unpublish`, `publications_for_view`, `get_definition`, `execute`).

Scope of this skill: `authoring_mode: "direct_sql"` only. `builder` mode exists (`builder.sources`) but is not covered — if asked, say so. The generic runtime view-query surface (`/v3/views/...`) is excluded by the doc (§13); use `execute`.

## Prerequisites

- The Pivotly MCP server is connected and authorized. Every underlying REST call carries `Authorization: Bearer <access_token>`; the server supplies it. Never ask the user for a token and never put one in a call.
- Permissions are `data_view.<action>`: `list`, `view`, `create`, `change` (update **and** unpublish), `delete`, `publish`, `run`. A missing permission returns **403** — report it, don't retry.
- Pre-flight once per session: call `pivotly_ping` before the first `config-item_data-view` call. If it fails, stop and tell the user the server is unreachable or not authorized.
- Reference the tools exactly as the server lists them: `config-item_data-view` and `pivotly_ping`. Clients that namespace tools add their own prefix.
- The SQL must read from a **published domain's** filtered read view, `usdf.<domain>` (e.g. `usdf.clients`). To learn which domains and columns exist, use the domain tooling (`config-item_domain`, e.g. the manage-domain skill) — never guess column names.
- The REST doc's optional `X-Tenant-Id` header (on authoring writes) has no tool parameter — tenant context, if any, is the server's configuration (not documented).
- Never send: `id` on `create` (→ 400); `item_type` / `itemType` (not tool properties — the schema has `additionalProperties: false`, so unknown keys are rejected). The REST create/update body requires `data.item_type` = `data_view`; the tool has no property for it, so it is expected to supply it itself (not documented) — if a `create` / `update` returns a 400 naming `item_type`, report it rather than adding the key.

## Pivotly MCP → `pivotly_ping`

- **Purpose:** liveness probe — echoes a message and reports protocol version and server build.
- **Side effects:** none (read-only per its description; the snapshot has no `annotations`).
- **Inputs:** `message` (string, `maxLength` 500, optional).
- **Output:** not documented beyond the description. Nothing chains from it.
- **Errors → next action:** any failure → stop, report, do not proceed.
- **Example:** `{ "message": "manage-data-view preflight" }`

## Pivotly MCP → `config-item_data-view`

One tool, thirteen operations chosen by the required `operation` property. The schema has `additionalProperties: false` — send only the properties listed here.

### Operation map

| `operation`             | REST endpoint (doc §)                                  | permission          | side effects                |
| ----------------------- | ------------------------------------------------------ | ------------------- | --------------------------- |
| `create`                | `POST /v3/config-items/data_view` (§8.1)               | `data_view.create`  | ⚠️ creates the config item  |
| `update`                | `PATCH /v3/config-items/data_view/{id}` (§8.2)         | `data_view.change`  | ⚠️ new config-item version  |
| `delete_config_item`    | `DELETE /v3/config-items/data_view/{id}` (§8.3)        | `data_view.delete`  | ⚠️ soft-delete              |
| `get`                   | `GET /v3/config-items/data_view/{id}` (§8.4)           | `data_view.view`    | read                        |
| `get_by_slug`           | `GET /v3/config-items/data_view/by-slug/{slug}` (§8.5) | `data_view.view`    | read                        |
| `list`                  | `GET /v3/config-items/data_view` (§8.6)                | `data_view.list`    | read                        |
| `filter`                | `GET /v3/config-items?itemType=data_view` (§8.7)       | `data_view.list`    | read                        |
| `publish`               | `POST /v3/data-views/{slug}/publish` (§9.1)            | `data_view.publish` | ⚠️ builds the runnable view |
| `unpublish`             | `DELETE /v3/data-views/{slug}/unpublish` (§9.2)        | `data_view.change`  | ⚠️ drops the runnable view  |
| `list_published`        | `GET /v3/data-views/publications` (§9.3)               | `data_view.view`    | read                        |
| `publications_for_view` | `GET /v3/data-views/{slug}/publications` (§9.4)        | `data_view.view`    | read                        |
| `get_definition`        | `GET /v3/data-views/{slug}/definition` (§9.5)          | `data_view.view`    | read                        |
| `execute`               | `POST /v3/data-views/{slug}/execute` (§10.1)           | `data_view.run`     | runs the read-only query    |

### Inputs (mirrors the tool's `inputSchema`)

| name                                               | type                                                     | used by                                                                                                                                                              | notes                                                                                                                                                          |
| -------------------------------------------------- | -------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `operation`                                        | string enum — the 13 values above                        | all                                                                                                                                                                  | **required** (the only schema-required property)                                                                                                               |
| `id`                                               | string                                                   | `update`, `delete_config_item`, `get` (required); `filter` (optional)                                                                                                | config-item uuid                                                                                                                                               |
| `slug`                                             | string                                                   | **required** for `create` **and** `update`, and for `get_by_slug`, `publish`, `unpublish`, `publications_for_view`, `get_definition`, `execute`; `filter` (optional) | must match `^[a-z][a-z0-9_-]{0,127}$` (malformed → 400 on `create`; `INVALID_SLUG` at publish)                                                                                                    |
| `cfg_data`                                         | object                                                   | **required** for `create` **and** `update`                                                                                                                           | the manifest — see Shared mechanics                                                                                                                            |
| `name`                                             | string                                                   | `create` / `update` (≤255); `filter`                                                                                                                                 |                                                                                                                                                                |
| `description`                                      | string \| null                                           | `create` / `update` (≤1000); `filter`                                                                                                                                |                                                                                                                                                                |
| `version`                                          | integer                                                  | `create` / `update`; `filter`                                                                                                                                        | config-item version                                                                                                                                            |
| `enabled`                                          | boolean                                                  | `create` / `update` (default `true`); `filter`                                                                                                                       |                                                                                                                                                                |
| `fetchType`                                        | `paginated` \| `list`                                    | `list`                                                                                                                                                               |                                                                                                                                                                |
| `page`                                             | number                                                   | `list`, `list_published`, `publications_for_view` (zero-based, default 0); `filter` (≥1)                                                                             | base differs by operation                                                                                                                                      |
| `pageSize`                                         | integer                                                  | `list` (1–100); `list_published`, `publications_for_view` (1–500, default 20)                                                                                        |                                                                                                                                                                |
| `slugContains`                                     | string                                                   | `list`, `list_published`                                                                                                                                             | substring on slug                                                                                                                                              |
| `search`                                           | string                                                   | `list`, `list_published`                                                                                                                                             | free text                                                                                                                                                      |
| `sortModel`                                        | array of `{ field, sort }`                               | `list`, `list_published`, `publications_for_view`                                                                                                                    | `sort` ∈ `asc` \| `desc`; both keys required                                                                                                                   |
| `filterModel`                                      | `{ items: [{ field, operator, value }], logicOperator }` | `list`, `list_published`, `publications_for_view`                                                                                                                    | `items`, `field`, `operator` required; `logicOperator` ∈ `and` \| `or`                                                                                         |
| `fullPath`, `createdBy`, `modifiedBy`, `isDeleted` | string \| number \| boolean                              | `filter`                                                                                                                                                             |                                                                                                                                                                |
| `limit`                                            | number                                                   | `filter`                                                                                                                                                             | ≤100                                                                                                                                                           |
| `parameters`                                       | object (name → value)                                    | `execute` only                                                                                                                                                       | keys **including** the `p_` prefix, matching `^[a-z][a-z0-9_]*$`; take names from `get_definition`. Not the same thing as `cfg_data.parameters` (see Pitfalls) |

**Result shape (all operations):** the snapshot has no `outputSchema`, so the REST envelope (Shared mechanics) is the guide — read the result before chaining a value. How the tool delivers a failure (an `isError` result vs the envelope with `error: true`) is not documented — surface `message` (and `details` when present) and ask before retrying.

### `create`

- **Purpose:** create a new data-view config item.
- **Side effects:** ⚠️ writes configuration. Confirm the payload first. Nothing is runnable until `publish`.
- **Inputs:** `slug`, `cfg_data` (required); optional `name`, `description`, `enabled`, `version`. Never `id`.
- **Output (201):** `data: { id, slug, version }` — keep `data.id` for `update` / `delete_config_item` / `get`.
- **Errors → next action:** 400 (missing/malformed `slug` — it must match the slug format — or `cfg_data`, `item_type` mismatch, id on create) → fix and resend; 403 → stop; 5xx duplicate slug → `get_by_slug`, then offer ⚠️ `update` or another slug. Manifest problems usually surface later, at `publish`.
- **Example:**

```json
{
  "operation": "create",
  "slug": "active-clients",
  "name": "Active Clients",
  "cfg_data": {
    "authoring_mode": "direct_sql",
    "implementation": "function",
    "direct_sql": "SELECT id, name FROM usdf.clients WHERE region = ${p_region}",
    "parameters": [{ "name": "p_region", "type": "text", "required": true }]
  }
}
```

→ `201` · `data: { "id": "…", "slug": "active-clients", "version": 1 }`

### `update`

- **Purpose:** change an existing manifest (SQL, parameters, name). Does **not** republish.
- **Side effects:** ⚠️ writes configuration; the runnable view is unchanged until the next `publish`. Confirm the diff.
- **Inputs:** `id`, `slug`, `cfg_data` (all required by the tool — send the **complete** `cfg_data`, a convention of this skill since the doc states merge semantics but not whether the merge descends into `cfg_data`); optional `name`, `description`, `enabled`.
- **Output (200):** `data: { id, slug, version }`, version incremented.
- **Errors:** 400 (body id ≠ path id, item-type mismatch); 403; 404 (id not found → look it up again); 5xx platform-state → read `message`.
- **Example:** `{ "operation": "update", "id": "…", "slug": "active-clients", "cfg_data": { "...": "the complete manifest" } }`

### `delete_config_item`

- **Purpose:** soft-delete the config item. **Side effects:** ⚠️ does not drop a published view function — `unpublish` first if it should stop running.
- **Inputs:** `id`. **Output (200):** `data: { id, slug, version }`. **Errors:** 403, 404.
- **Example:** `{ "operation": "delete_config_item", "id": "…" }`

### `get` / `get_by_slug`

- **Purpose:** fetch the full config item by `id` or `slug`. **Side effects:** none.
- **Output (200):** **camelCase** response: `{ id, slug, itemType, version, name, enabled, cfgData }`. `id` feeds `update`; `cfgData` is the manifest to edit — send it back under the key `cfg_data`.
- **Errors:** 403; 404 (in a create flow, 404 from `get_by_slug` means the slug is free).
- **Example:** `{ "operation": "get_by_slug", "slug": "active-clients" }`

### `list` / `filter`

- **Purpose:** find data-view config items. **Side effects:** none.
- **`list` inputs:** `fetchType`, `page` (zero-based), `pageSize` (1–100), `slugContains`, `search`, `sortModel`, `filterModel`. **Output:** `data` = config items + `pagination`.
- **`filter` inputs:** at least one of `id`, `version`, `enabled`, `name`, `description`, `slug`, `fullPath`, `createdBy`, `modifiedBy`, `isDeleted`; paging `page` (≥1), `limit` (≤100). No filter → 400. The endpoint's `itemType=data_view` is supplied by the tool.
- **Example:** `{ "operation": "list", "slugContains": "client", "sortModel": [{ "field": "slug", "sort": "asc" }] }`

### `publish`

- **Purpose:** validate the saved manifest and compile it into a runnable, parameterized view function. **This is the real validator** — run the pre-publish checklist (Shared mechanics) first.
- **Side effects:** ⚠️ builds / replaces the runnable view. Confirm.
- **Inputs:** `slug`. No body.
- **Output (200):** `data: { slug, db_object_name, pub_id }` and a `message` like `Data view "active-clients" published as function …` — surface it.
- **Errors → next action:** 404 `NOT_FOUND` → no config for that slug; check `list`. 403 → stop. 5xx with a code in `message` → see the error table in Shared mechanics; fix the manifest via ⚠️ `update`, then ⚠️ re-publish.
- **Example:** `{ "operation": "publish", "slug": "active-clients" }`

### `unpublish`

- **Purpose:** drop the published view function; the config item stays.
- **Side effects:** ⚠️ `execute` stops working until re-published. Confirm.
- **Inputs:** `slug`. **Output (200):** `message: "Data view unpublished"`.
- **Errors:** 403; 5xx `UNPUBLISH_FAILED` (report `message`), `MISSING_PARAM` (slug missing).
- **Example:** `{ "operation": "unpublish", "slug": "active-clients" }`

### `list_published` / `publications_for_view`

- **Purpose:** what is live, and one view's publication history. **Side effects:** none.
- **`list_published` output:** rows `{ view_slug, status, implementation, db_object_name, published_at, published_by, name, description }` + `pagination` (one row per slug: its latest complete function publication).
- **`publications_for_view` output:** rows `{ id, view_slug, status, implementation, parameter_signature, db_object_name, published_at, published_by }` + `pagination`. `status: "complete"` = executable; other states are in-progress/failed (exact values not documented).
- **Example:** `{ "operation": "publications_for_view", "slug": "active-clients", "sortModel": [{ "field": "published_at", "sort": "desc" }] }`

### `get_definition`

- **Purpose:** the publish-time snapshot — the **authoritative parameter names and types** to use at `execute`, plus output columns.
- **Side effects:** none. **Inputs:** `slug`.
- **Output (200):** `data: { db_object_name, parameter_signature, view_cfg: { parameters, columns } }`. `columns` may be an empty array — don't assume it is populated.
- **Errors:** 404 (no published function → publish first); 403.
- **Example:** `{ "operation": "get_definition", "slug": "active-clients" }`

### `execute`

- **Purpose:** run a published view with named parameters and return rows.
- **Side effects:** none beyond running the read-only query.
- **Inputs:** `slug`; `parameters` — object keyed by the exact names from `get_definition` (`p_` prefix included), values in their JSON types (`"APAC"`, `42`, `true`). Omit or `{}` for a view without parameters.
- **Output (200):** `data` = array of result rows.
- **Errors → next action:** 404 → no `status=complete`, `implementation=function` publication ("Publish the data view first.") → offer ⚠️ `publish`; 400 → a key is not a valid identifier (check spelling/prefix against `get_definition`); 403 → stop; 5xx → the query failed at run time; report `message` (often a value that doesn't fit the parameter's `type`).
- **Example:** `{ "operation": "execute", "slug": "active-clients", "parameters": { "p_region": "APAC" } }` → `data: [{ "id": "…", "name": "…" }, …]`

## Shared mechanics

### Response envelope (doc §4)

Success: `{ data, meta, status, error: false, message?, pagination? }`. Error: `{ data: null, meta: {}, status, error: true, message, details? }`. Branch on `error`, then `message` / `details`. `pagination` = `{ page, page_size, total_records }`.

### Spelling layers

Request properties are snake_case (`cfg_data`, `direct_sql`, `authoring_mode`), GET config-item responses are camelCase (`itemType`, `cfgData`), publication/definition responses are snake_case (`view_slug`, `db_object_name`, `parameter_signature`, `view_cfg`, `pub_id`). Query/filter tool properties are camelCase (`pageSize`, `slugContains`, `fullPath`, `isDeleted`). Copying `cfgData` from a GET into an update means sending it as `cfg_data`. In REST, `sortModel` / `filterModel` are JSON-encoded query strings; the tool takes the structured array / object.

`filterModel.items[].operator` ∈ `contains`, `doesNotContain`, `equals`, `doesNotEqual`, `startsWith`, `endsWith`, `isEmpty`, `isNotEmpty`, `is`, `not`, `isAnyOf`, `=`, `!=`, `>`, `>=`, `<`, `<=`, `after`, `before`, `onOrAfter`, `onOrBefore`.

### `cfg_data` — the direct-SQL manifest (doc §6)

```jsonc
{
  "authoring_mode": "direct_sql", // "direct_sql" (default) | "builder" — this skill always sends "direct_sql"
  "implementation": "function", // "function" (default, executable) | "view" (NOT executable via execute)
  "direct_sql": "SELECT id, name FROM usdf.clients WHERE region = ${p_region}", // REQUIRED in direct_sql mode; read-only SELECT
  // "builder": { "sources": [...] } — REQUIRED only in builder mode (out of scope)
  "parameters": [
    {
      "name": "p_region", // REQUIRED; ^p_[a-z][a-z0-9_]*$ — p_ prefix mandatory, stored verbatim
      "type": "text", // text (default) | integer | uuid | boolean | numeric | date | timestamptz; "data_type" accepted as alias key
      "default": null, // optional
      "required": false // optional boolean, default false
    }
  ],
  "area": "sales", // optional; normalized on publish
  "columns": [] // optional; output columns are introspected at publish
}
```

Convention of this skill (not system defaults): always send `authoring_mode: "direct_sql"` and `implementation: "function"` explicitly, and always send `type` on every parameter (not `data_type`, and not both).

### Parameter references — rules

| #   | Rule                                                                                                                                                                                                                                                                                                  | Source                                             |
| --- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------- |
| 1   | Reference a parameter in `direct_sql` as `${p_<name>}` — the `parameters[].name`, character for character, inside `${` `}`: `${p_region}`. Not bare `p_region`, not `$p_region`, not `{p_region}`. | doc §6 |
| 2   | Names match `^p_[a-z][a-z0-9_]*$`: lowercase, starts `p_` + a letter, then letters/digits/underscores. `region`, `p_Region`, `p-region`, `p_1st` are all wrong. The `${…}` wrapper belongs only in the SQL, never in `parameters[].name`. | doc §6/§7 |
| 3   | Publish does **not** add the prefix. A parameter named `region` → `INVALID_PARAMETER_NAME_PREFIX`; a parameter with no `name` → `INVALID_CFG_DATA`.                                                                                                                                                   | doc §6/§11                                         |
| 4   | At `execute`, pass the same key (`"p_region"`), taken from `get_definition` — never the name without its `p_` prefix.                                                                                                                                                                                 | doc §10.1                                          |
| 5   | A reference stands where a **value** goes (as in `WHERE region = ${p_region}`). Don't wrap it in quotes (`'${p_region}'`) or concatenate it into strings. Use references for values only, never for table, column, or keyword names. | convention |
| 6   | Every `${p_…}` reference in `direct_sql` has a matching `parameters[]` entry, and every declared parameter is used. No bare `p_…` name is left where a parameter was meant (a bare name is not the documented reference form). The doc does not say what publish does with a mismatch — check it yourself before publishing. | convention |
| 7   | Pick `type` to match the column being compared (`integer` for integer ids, `uuid` for uuid ids, `date` / `timestamptz` for dates). Execute-time values arrive in their JSON types; the accepted string format for `date` / `timestamptz` is not documented — read `message` on the first 5xx.         | doc §6/§10.1                                       |
| 8   | A non-required parameter with no `default` binds as `DEFAULT NULL`. In SQL, `col = NULL` matches no rows — for an optional filter write `(${p_region} IS NULL OR region = ${p_region})`. | doc §6 (NULL binding); SQL pattern is a convention |
| 9   | A `${…}` expression `default` also binds as `DEFAULT NULL` for direct callers. Which expressions a default may contain is not documented — don't invent one; if the user wants one, store it as they give it, warn that callers may receive NULL, and write the SQL to tolerate NULL (rule 8). | doc §6 |
| 10  | `required: true` means the caller must supply it; still pass every parameter explicitly at `execute` rather than relying on defaults.                                                                                                                                                                 | convention                                         |
| 11  | Never put a user-supplied value directly into `direct_sql`; declare a parameter instead.                                                                                                                                                                                                              | safety policy                                      |

### SQL rules (enforced at publish)

- Read-only `SELECT` only — any DDL / non-SELECT keyword → `INVALID_SQL_DDL_KEYWORD`. If this fires on what you believe is a plain SELECT, look for a keyword-like word elsewhere in the text (an alias, a comment) — the doc doesn't say how the check scans.
- Read from `usdf.<domain>` (the filtered read view, which applies access and system gating) — never `usdf.<domain>_b` (base) or `usdf.<domain>_c` (archive) → `INVALID_SQL_BASE_TABLE_REFERENCE`.
- `direct_sql` non-empty; `authoring_mode` / `implementation` valid → else `INVALID_CFG_DATA`.
- `slug` matches `^[a-z][a-z0-9_-]{0,127}$` → else 400 on `create`, `INVALID_SLUG` at publish.

### Pre-publish checklist

1. `authoring_mode: "direct_sql"`, `implementation: "function"` (unless the user explicitly wants a non-executable `view`).
2. SQL is a single read-only `SELECT` over `usdf.<domain>` names only.
3. Find every `${…}` reference in `direct_sql`; each is `${p_[a-z][a-z0-9_]*}` and its inner name appears in `parameters[].name`; no declared parameter is unused; no bare `p_…` name is left where a parameter was meant.
4. Every parameter has a valid `type`; optional parameters are NULL-safe in the SQL.
5. No literal user value is embedded in the SQL.

### Error codes (doc §11)

| Code / condition                   | HTTP | Where                    | Next action                                                                     |
| ---------------------------------- | ---- | ------------------------ | ------------------------------------------------------------------------------- |
| request validation                 | 400  | create / update / delete | fix the request shape (missing/mis-typed field, malformed slug, id on create, id/type mismatch)                                                           |
| duplicate slug / platform-state    | 5xx  | create / update          | read `message`; `get_by_slug` → ⚠️ `update`, or new slug                        |
| `NOT_FOUND`                        | 404  | publish                  | no config for slug; check `list`                                                |
| `MISSING_PARAM`                    | 5xx  | publish, unpublish       | slug missing                                                                    |
| `INVALID_SLUG`                     | 5xx  | publish                  | slug format                                                                     |
| `INVALID_CFG_DATA`                 | 5xx  | publish                  | bad mode/implementation, empty `direct_sql`, missing parameter name, bad `type` |
| `INVALID_PARAMETER_NAME_PREFIX`    | 5xx  | publish                  | rename to `p_…` in `parameters[]` **and** every `${…}` reference in `direct_sql`       |
| `INVALID_SQL_DDL_KEYWORD`          | 5xx  | publish                  | make it a pure SELECT                                                           |
| `INVALID_SQL_BASE_TABLE_REFERENCE` | 5xx  | publish                  | `usdf.<domain>_b` / `_c` → `usdf.<domain>`                                      |
| `INVALID_FILTER_OP`                | 5xx  | publish                  | builder mode only — out of scope                                                |
| `PUBLISH_FAILED`                   | 5xx  | publish                  | generic; report `message`, stop                                                 |
| `UNPUBLISH_FAILED`                 | 5xx  | unpublish                | report `message`                                                                |
| no published function              | 404  | execute, get_definition  | ⚠️ publish first                                                                |
| invalid parameter key              | 400  | execute                  | key must match `^[a-z][a-z0-9_]*$`; use `get_definition` names                  |
| execution failure                  | 5xx  | execute                  | read `message` (often a type mismatch)                                          |
| not authorized                     | 403  | all                      | stop; missing `data_view.<action>`                                              |

## Workflows

### A. Create and publish a new data view

1. `pivotly_ping` (once per session).
2. `get_by_slug` with the intended slug — 404 = free; 200 = use Workflow B.
3. Confirm the source domain and its columns (domain tooling); never guess them.
4. Write `direct_sql` with a `${p_<name>}` reference for every per-run value; declare each in `parameters[]` with `type` and `required`. Run the pre-publish checklist.
5. ⚠️ Show slug, SQL and parameters; on yes → `create`. Keep `data.id`.
6. ⚠️ Confirm → `publish`. Surface `message`. On a 5xx code, fix via ⚠️ `update` (full `cfg_data`) and ⚠️ re-publish.
7. `get_definition` → report the parameter signature and columns. Offer a first `execute` with values the user chooses.

### B. Change an existing view (edit SQL, add/rename/make optional a parameter)

1. `get_by_slug` → `id`, `slug`, `cfgData`.
2. Edit a copy of `cfgData`. Renaming a parameter means changing `parameters[].name` **and** every `${p_…}` reference in `direct_sql` together. If the stored SQL still uses bare `p_…` names (older skill convention), convert them to `${p_…}` in the same edit and say so. Making one optional means `required: false` plus NULL-safe SQL (rule 8).
3. Run the pre-publish checklist.
4. ⚠️ Show the diff → `update` with `id`, `slug`, complete `cfg_data`.
5. ⚠️ Confirm → `publish` (the old function keeps running until then). Tell the user that callers must use the new parameter names after a rename.

### C. Run a view

1. `get_definition` → exact names and types in `view_cfg.parameters` / `parameter_signature`.
2. Map the user's words to those keys (e.g. "for APAC" → `p_region: "APAC"`); ask for any required value you don't have.
3. `execute` with `slug` + `parameters`. On 404 → offer ⚠️ `publish`; on 400 → fix key spelling; on 5xx → report `message`.

### D. Status questions

What's live → `list_published`; history for one view → `publications_for_view` (newest first via `sortModel` on `published_at`); what it takes and returns → `get_definition`; stored manifest → `get_by_slug`.

### E. Retire a view

1. Ask which is meant: stop it running (`unpublish`), remove the config item (`delete_config_item`), or both.
2. ⚠️ Confirm → `unpublish` with `slug`.
3. ⚠️ Confirm → `delete_config_item` with `id` (from `get_by_slug`). Ordering is this skill's convention.

## User phrasing → call

| user says                                                                            | call                                                                |
| ------------------------------------------------------------------------------------ | ------------------------------------------------------------------- |
| "create a data view that returns X by Y", "new parameterized query over clients"     | Workflow A → ⚠️ `create`                                            |
| "add a region filter", "add a parameter", "make the date optional", "change the SQL" | Workflow B → `get_by_slug`, ⚠️ `update`, ⚠️ `publish`               |
| "my parameter isn't substituted", "do I need ${}", "INVALID_PARAMETER_NAME_PREFIX" | check parameter-reference rules 1–6 (every reference is `${p_…}`) → Workflow B |
| "publish the view", "make it live", "apply my changes"                               | ⚠️ `publish` `{ slug }`                                             |
| "run active-clients for APAC", "execute the view", "get me the rows"                 | Workflow C → `get_definition`, `execute`                            |
| "what parameters does X take", "what columns does X return"                          | `get_definition` `{ slug }`                                         |
| "which views are published / live"                                                   | `list_published`                                                    |
| "when was X last published", "did the publish work"                                  | `publications_for_view` `{ slug }`                                  |
| "list data views", "find views with 'client' in the slug"                            | `list` (+ `slugContains`, `search`)                                 |
| "show me the SQL for X"                                                              | `get_by_slug` → `cfgData.direct_sql`                                |
| "it says publish the data view first"                                                | ⚠️ `publish`, then retry `execute`                                  |
| "rejected for clients_b"                                                             | rewrite to `usdf.clients` → Workflow B                              |
| "stop the view", "unpublish X"                                                       | ⚠️ `unpublish` `{ slug }`                                           |
| "delete the data view"                                                               | Workflow E (ask which delete)                                       |
| "is Pivotly up"                                                                      | `pivotly_ping`                                                      |

## Pitfalls

- **Two different `parameters`.** `cfg_data.parameters` is the declaration array (`[{ name, type, default, required }]`) used on `create` / `update`; the top-level tool property `parameters` is the execute-time value object (`{ "p_region": "APAC" }`). Don't send one where the other belongs.
- **Prefix in all three places.** Declaration `p_region`, SQL `${p_region}`, execute key `p_region` (no `${}` outside the SQL). The execute endpoint's own regex (`^[a-z][a-z0-9_]*$`) would accept `region`, but the published function has no such argument — always use `get_definition` names.
- **Reference syntax changed in this revision.** Earlier versions of this skill wrote bare `p_region` in `direct_sql`; the reference doc (rev 2026-09-23) specifies `${p_region}`. Views already saved with bare names: how publish treats them is not documented — when editing one, migrate it (Workflow B) and re-publish.
- **Save ≠ publish.** `create` / `update` succeed even with a broken manifest; errors appear at `publish`. After an `update`, the old function keeps running until the next `publish`.
- **Ordinary rejections arrive as HTTP 5xx** (every publish validation code, duplicate slug, `UNPUBLISH_FAILED`). Branch on `error` / `message`, never on status alone; don't auto-retry. Reliable 4xx: request shape (400), publish `NOT_FOUND`, execute 404/400, 403.
- **`implementation: "view"` is not executable** via `execute` — you get 404. Use `function` unless the user explicitly wants otherwise.
- **Optional parameters become NULL.** A non-required parameter without a default, and any expression default, binds as `DEFAULT NULL`; `col = p_x` then returns nothing. Write NULL-safe predicates.
- **Base/archive tables are rejected.** `usdf.clients_b` / `usdf.clients_c` → use `usdf.clients`.
- **`columns` may be empty** in `get_definition` — describe results from the rows instead.
- **`delete_config_item` does not stop the view** — `unpublish` does.
- **Not documented — kept explicit:** the tool's exact result / failure delivery (`isError` vs envelope); in-progress/failed `status` values; which expressions a `default` may contain; accepted string formats for `date` / `timestamptz` values; publish behavior when SQL references and declared parameters don't match. Read results before chaining; ask before retrying.
- **Out of scope:** `builder` mode (`builder.sources` — missing it → `INVALID_CFG_DATA`; `INVALID_FILTER_OP`) and the `/v3/views/...` runtime query surface (§13).
