---
name: manage-domain
description: Author, publish, inspect and delete Pivotly domains (governed, versioned data entities managed as config items) through the Pivotly MCP tool config-item_domain. Use it whenever someone mentions a Pivotly domain or a domain config item, or says things like 'create a domain', 'set up a new domain for customers', 'add a column to the products domain', 'change a field's data type', 'add an index', 'give this column a default value', 'bind a picklist to this field', 'set up change tracking for a system', 'publish the domain', 'did the publish go through', 'why did my domain save fail', 'validation failed on cfg_data', 'Schema validation error on update', 'list our domains', 'show me the domain config', 'what columns does that domain have', 'domain metadata', 'publication history', 'soft-delete the config item', 'delete the published domain', 'drop the domain', 'rename a domain', 'track_versions', 'xref_enabled', 'domain_table_name' - even when Pivotly isn't named.
category: core
---

```skill
category: core
```

# Manage Pivotly Domains

## Safety / confirmation policy

Applies to every call in this skill, not only inside the workflows.

| Operation(s) | Class | Rule |
| --- | --- | --- |
| `get`, `get_by_slug`, `list`, `filter`, `list_publications`, `list_published`, `get_publication`, `publications_for_domain`, `metadata_by_version`, `list_metadata`; `pivotly_ping` | read-only | Call freely; answer questions from these before touching anything. |
| `create` | ⚠️ writes configuration | Show the exact fields / `cfg_data` you will send and get an explicit yes first. Saving never changes live data structures. |
| `update` | ⚠️ writes configuration — and **upserts** | Show the diff and get a yes. Only ever use an `id` you just read back from `get_by_slug` / `get` / `list`: an unknown `id` does **not** fail, it silently **creates a new config item at that id**. `cfg_data` is replaced wholesale, so send the complete current model with your edits. |
| `publish` | ⚠️ changes live data structures | Confirm the slug and that the latest **saved** configuration is what they want live. Warn when `track_versions` or `xref_enabled` is `true`: once published `true`, neither can be set back to `false`. |
| `delete_config_item` | ⚠️ soft-deletes the configuration record | Confirm slug and id. Does **not** drop a published domain's live data structures. |
| `delete_published` | ⚠️⚠️ irreversible | Permanently deletes the published domain and **all** of its data structures and metadata. Require two explicit confirmations, the second repeating the slug; check `publications_for_domain` first; never fold it into a broader "clean up" step. |

Never guess a slug, id or column name — look it up (`get_by_slug`, `list`). Never retry a mutating call on a 5xx without reading `message` / `details` first (several ordinary rejections arrive as 5xx — see Pitfalls). When the user says "delete the domain", ask which delete they mean unless context makes it clear.

## Mental model

A **domain** is a governed, versioned data entity. You author it as a **config item** — a JSON configuration record with `item_type` = `domain`, identified by a `slug` and a uuid `id`. The domain model lives in `cfg_data`: one `schema[]` entry per column, plus conflict-resolution, archive, versioning, index and per-system settings.

Authoring and publishing are separate steps. `create` / `update` only store configuration (each save bumps the config-item `version`, which the platform sets — you never send it); nothing changes live until `publish`, which runs **publish → prune → archive** and materializes or refreshes the data structures. Publication runs are inspectable (`list_publications`, `get_publication`, `publications_for_domain`, `list_published`) and the live per-domain-version metadata (columns, types, flags, masks, tenants) comes from `metadata_by_version` / `list_metadata`.

`update` is a **full resubmission**, not a partial patch: every update carries the current `slug` and the complete `cfg_data`, which replaces the stored model wholesale. The scalar fields (`name`, `description`, `enabled`, `parent_item_id`, `full_path`) patch-merge — omitted ones keep their stored value.

Retiring has two distinct operations: `delete_config_item` soft-deletes the configuration record; `delete_published` permanently drops the published domain and its data.

Identity by operation: config-item `id` (uuid) → `update`, `delete_config_item`, `get`; `slug` → `create`, `update` (the current slug), `get_by_slug`, `publish`; `domain` (the same slug, once published) → `publications_for_domain`, `metadata_by_version`, `delete_published`; publication-run `id` → `get_publication`.

Out of scope (doc §13): reading or writing records inside a published domain, and data-model/ERD endpoints. Say so if asked — this tool does not do them.

## Prerequisites

- The Pivotly MCP server is connected and authorized. Every underlying REST call carries `Authorization: Bearer <access_token>`; the server supplies it. Never ask the user for a token and never put one in a call. How the server is configured with credentials is not documented.
- Permissions are `domain.<action>`: `list`, `view`, `create`, `change`, `delete`, `publish`. A missing permission returns **403** — report it, don't retry.
- Pre-flight once per session: call `pivotly_ping` before the first `config-item_domain` call. If it fails, stop and tell the user the server is unreachable or not authorized.
- Reference the tools exactly as the server lists them: `config-item_domain` and `pivotly_ping`. Clients that namespace tools add their own prefix.
- The REST doc's optional `X-Tenant-Id` header has no tool parameter — tenant context, if any, is the server's configuration (not documented).
- Never send: an `id` on `create` (→ 400); `item_type` or `itemType` (not tool properties — the tool is fixed to `domain`, and the schema has `additionalProperties: false`, so unknown keys are rejected); `version` on `create` / `update` (ignored — the platform sets it); `cfg_data.access_control` (server-injected on every save, overwritten); `domain_name` (not a recognized field); reserved or `sys_` / `_` / `p_`-prefixed column names; the flat `picklist_slug` / `picklist_parameters` / `picklist_source_type` keys.

## Pivotly MCP → `pivotly_ping`

- **Purpose:** liveness probe — echoes a message back and reports the negotiated protocol version and server build. Confirms connectivity and credentials before anything else.
- **Side effects:** none (read-only per its description; the snapshot has no `annotations`).
- **Inputs**

| name | type | required | notes |
| --- | --- | --- | --- |
| `message` | string, `maxLength` 500 | no (schema has no `required` list) | text to echo back |

- **Output:** not documented beyond the description (echo, protocol version, server build). Nothing chains from it.
- **Errors → next action:** not documented. Any failure → stop, report, do not proceed to domain calls.
- **Example:** `{ "message": "manage-domain preflight" }`

## Pivotly MCP → `config-item_domain`

One tool, fifteen operations chosen by the required `operation` property; each operation is one REST endpoint from the doc's coverage checklist (§12). The schema has `additionalProperties: false` — send only the properties listed here.

### Operation map

| `operation` | REST endpoint (doc §) | permission | side effects |
| --- | --- | --- | --- |
| `create` | `POST /v3/config-items/domain` (§8.1) | `domain.create` | ⚠️ creates the config item |
| `update` | `PATCH /v3/config-items/domain/{id}` (§8.2) | `domain.change` | ⚠️ new config-item version; upserts on unknown id |
| `delete_config_item` | `DELETE /v3/config-items/domain/{id}` (§8.3) | `domain.delete` | ⚠️ soft-delete |
| `get` | `GET /v3/config-items/domain/{id}` (§8.4) | `domain.view` | read |
| `get_by_slug` | `GET /v3/config-items/domain/by-slug/{slug}` (§8.5) | `domain.view` | read |
| `list` | `GET /v3/config-items/domain` (§8.6) | `domain.list` | read |
| `filter` | `GET /v3/config-items?itemType=domain` (§8.7) | `domain.list` | read |
| `publish` | `POST /v3/domain-publish/{slug}` (§9.1) | `domain.publish` | ⚠️ changes live data structures |
| `list_publications` | `GET /v3/domain-publications` (§9.2) | `domain.view` | read |
| `list_published` | `GET /v3/domain-publications/published` (§9.3) | `domain.view` | read |
| `get_publication` | `GET /v3/domain-publications/{id}` (§9.4) | `domain.view` | read |
| `publications_for_domain` | `GET /v3/domain-publications/domain/{domain}` (§9.5) | `domain.view` | read |
| `metadata_by_version` | `GET /v3/domain-info/{domain}/{version}` (§9.6) | `domain.view` | read |
| `list_metadata` | `GET /v3/domain-info` (§9.7) | `domain.list` | read |
| `delete_published` | `DELETE /v3/domain/{domain}` (§10.1) | `domain.delete` | ⚠️⚠️ irreversible |

### Inputs (mirrors the tool's `inputSchema`)

| name | type | used by | notes |
| --- | --- | --- | --- |
| `operation` | string enum — the 15 values above | all | **required** (the only schema-required property) |
| `id` | string | `update`, `delete_config_item`, `get` (config-item uuid); `filter` (optional column); `get_publication` (publication-run id) | overloaded — two kinds of id |
| `slug` | string | `create` (required); `update` (required — the **current** slug; a different value renames, allowed only while unpublished); `get_by_slug`, `publish`; `filter` (optional column) | 1–50 chars; must match `^[a-z0-9](?:[a-z0-9_]*[a-z0-9])?$` (lowercase, digits, underscores; **no dashes**; no leading/trailing `_`); must not start with `sys_`, `dvw_`, `dtm_`, `dtv_`; unique across config items |
| `domain` | string | `publications_for_domain`, `metadata_by_version`, `delete_published` | the published domain's slug |
| `cfg_data` | object | `create` (required); `update` (required — the **complete** model, replaced wholesale) | the domain model — shape in Shared mechanics |
| `name` | string | `create` / `update`; `filter` (optional column) | ≤255; defaults to slug on create; patch-merged on update |
| `description` | string \| null | `create` / `update`; `filter` (optional column) | ≤1000; on update send `null` to clear, omit to keep |
| `version` | integer | `metadata_by_version` (domain version, positive); `filter` (optional column) | **ignored** on `create` (forced to 1) and `update` (stored + 1, no concurrency check) — don't send it there |
| `enabled` | boolean | `create` / `update`; `filter` (optional column) | default `true` on create |
| `parent_item_id` | string | `create` / `update` | uuid |
| `full_path` | string | `create` / `update` | ≤500; defaults to slug on create |
| `fetchType` | `paginated` \| `list` | `list` | default `paginated`; `list` returns the full set with `page` / `page_size` = `null` |
| `page` | integer | `list` (zero-based, default 0); `list_metadata` (≥0); `filter` (**one-based**, ≥1, default 1) | base differs by operation |
| `pageSize` | integer | `list` (default 25), `list_metadata` | 1–100 |
| `slugContains` | string | `list` | substring match on slug |
| `search` | string | `list` (free text); `list_metadata` (substring on slug) | |
| `sortModel` | array of `{ field, sort }` | `list` | `sort` ∈ `asc` \| `desc`; both keys required per entry |
| `filterModel` | `{ items: [{ field, operator, value }], logicOperator }` | `list` | operators in Shared mechanics; `logicOperator` ∈ `and` \| `or`; `items`, `field`, `operator` required |
| `fullPath` | string | `filter` | same concept as the body's `full_path` — different layer, different spelling |
| `depth` | integer | `filter` | REST types it `number` (coerced, not integer-constrained) |
| `createdBy` | string | `filter` | |
| `modifiedBy` | string | `filter` | |
| `parentItemId` | string | `filter` | body spelling is `parent_item_id` |
| `isDeleted` | boolean | `filter` | |
| `limit` | integer | `filter` | ≤100, default 25; no enforced minimum — send ≥1 |

**Result shape (all operations):** the tool snapshot has no `outputSchema`, so the REST envelope (Shared mechanics) is the best guide — read the result before chaining a value into the next call. Success vs failure is the boolean `error`, not the HTTP status. How the tool delivers a failure (an `isError` result vs the envelope with `error: true`) is not documented — surface `message` (and `details` when present) to the user and ask before retrying.

### `create`

- **Purpose:** create a new domain config item (create-only; later changes go through `update`).
- **Side effects:** ⚠️ creates the configuration record at `version` 1. Confirm the payload first.
- **Inputs:** `slug` (required), `cfg_data` (required — complete and valid: every validator-required field), optional `name` (→ slug), `description`, `enabled` (→ `true`), `parent_item_id`, `full_path` (→ slug). Never `id`, never `version`.
- **Output (201):** `data: { id, slug, version }` — keep `data.id` for `update` / `delete_config_item` / `get`.
- **Errors → next action:**
  - 400 (request shape: malformed or oversize field, wrong JSON type, id sent on create) → fix and resend.
  - 403 → lacks `domain.create`; stop.
  - 5xx `VALUE` → slug longer than 50 or failing the slug regex (dashes, leading/trailing `_`); pick a valid slug.
  - 5xx, `message` "Validation failed" (`INVALID`) → top-level `details` (with `data: null`) is a per-path list of `{ path, code, detail, severity, remediation, proposed?, current? }`. Fix every entry with severity other than `warning` — `remediation` / `proposed` say how — then resend. Not a server fault.
  - 5xx, duplicate slug → `message` directs you to PATCH the existing id: `get_by_slug` for it, then `update` (after confirming) — or choose another slug.
- **Example:**

```json
{
  "operation": "create",
  "slug": "products",
  "name": "Products",
  "description": "Product master data",
  "enabled": true,
  "cfg_data": {
    "domain_name_singular": "Product",
    "domain_table_name": "products",
    "default_conflict_resolution": { "code": "lww", "human_readable": "Last Write Wins" },
    "archive": true,
    "archive_policy": { "max_record_versions": 10, "retention_days": 365 },
    "track_versions": true,
    "xref_enabled": true,
    "multi_tenant": false,
    "dedupe_enabled": true,
    "schema": [
      {
        "name": "product_number",
        "column_name": "product_number",
        "data_type": "text",
        "triggers_version_change": true,
        "description": "Unique product number (SKU)",
        "indexed": true,
        "unique": true
      }
    ]
  }
}
```

→ `201` · `data: { "id": "65eb81bf-0a01-405b-b443-ffe3e68bf48e", "slug": "products", "version": 1 }`

### `update`

- **Purpose:** save a changed domain config item at an existing id; produces a new config-item version.
- **Side effects:** ⚠️ writes configuration (live structures unchanged until `publish`). Confirm the diff first. ⚠️ **Upsert:** an `id` with no record creates a new config item at that id (200, version 1) instead of failing — only use an id just read from `get_by_slug` / `get` / `list`.
- **Inputs:** `id` (required — freshly looked up), `slug` (required — the current slug), `cfg_data` (required — the **complete** current model with your edits; it replaces the stored one wholesale, so anything you leave out is removed). Optional, patch-merged (omitted = kept): `name`, `description` (`null` clears), `enabled`, `full_path`, `parent_item_id`. Renaming: send a new `slug` only while the domain is unpublished, with `cfg_data.domain_table_name` changed to match.
- **Output (200):** `data: { id, slug, version }` with `version` incremented by 1.
- **Errors → next action:**
  - 400 `message` "Schema validation" → a required field (`slug`, `cfg_data`, or the tool-supplied `item_type`) is missing, oversize, or an id in the body differs from the path id. Add the missing field and resend. If it names `item_type`, the tool did not supply it — report that as a server-side gap; you cannot send it yourself.
  - 403 → lacks `domain.change`.
  - 5xx `INVALID` → same top-level `details` handling as `create`. On a published domain also expect the immutability codes (see Shared mechanics → detail codes).
  - 5xx `VALUE` → slug fails the length/regex rule.
  - 5xx with `IMMUTABLE_SLUG` in `message` → the domain is published; its slug cannot change. A rename means a new domain.
  - 5xx duplicate slug → the `slug` you sent belongs to another config item — often a sign the `id` was wrong. Re-read with `get_by_slug`.
  - 5xx item-type change (`TYPE_CHANGE_NOT_ALLOWED`) → the stored `item_type` cannot change; nothing in this tool should trigger it.
  - There is **no 404** on `update` — do not rely on one to detect a bad id.
- **Example (change only the description — still resend slug and the full current `cfg_data`):**

```json
{
  "operation": "update",
  "id": "65eb81bf-0a01-405b-b443-ffe3e68bf48e",
  "slug": "products",
  "description": "Updated product master data",
  "cfg_data": {
    "domain_name_singular": "Product",
    "domain_table_name": "products",
    "default_conflict_resolution": { "code": "lww", "human_readable": "Last Write Wins" },
    "schema": [
      { "name": "product_number", "column_name": "product_number", "data_type": "text", "triggers_version_change": true },
      { "name": "list_price", "column_name": "list_price", "data_type": "numeric", "triggers_version_change": false }
    ]
  }
}
```

→ `200` · `data: { "id": "…", "slug": "products", "version": 2 }`. The `cfg_data` must be the domain's current model from a GET, or the wholesale replace changes it.

### `delete_config_item`

- **Purpose:** soft-delete the configuration record (marks it deleted and disabled).
- **Side effects:** ⚠️ removes the config item from authoring; does **not** drop live data structures — that is `delete_published`.
- **Inputs:** `id` (required). Nothing else is needed; any body fields are patch-merged against the stored record.
- **Output (200):** `data: { id, slug, version }`.
- **Errors:** 403; 404 (unknown id).
- **Example:** `{ "operation": "delete_config_item", "id": "65eb81bf-0a01-405b-b443-ffe3e68bf48e" }`

### `get` / `get_by_slug`

- **Purpose:** fetch the full config item by uuid (`get`) or by slug (`get_by_slug`).
- **Side effects:** none.
- **Inputs:** `id` for `get`; `slug` for `get_by_slug`.
- **Output (200):** the config item, **camelCase** in the response: `data: { id, slug, itemType, version, name, enabled, cfgData }`. `data.id` feeds `update` / `delete_config_item`; `data.slug` feeds `update`'s `slug`; `data.cfgData` is the model to edit — send it back under the key `cfg_data`. It may contain the server-injected `access_control`; drop it before resending (it is overwritten anyway).
- **Errors:** 403; 404 (unknown id/slug — in a create flow, 404 from `get_by_slug` means the slug is free).
- **Example:** `{ "operation": "get_by_slug", "slug": "products" }`

### `list`

- **Purpose:** paginated, sortable, filterable list of domain config items (grid-style).
- **Side effects:** none.
- **Inputs (all optional):** `fetchType`, `page` (zero-based), `pageSize`, `slugContains`, `search`, `sortModel`, `filterModel`.
- **Output (200):** `data` = array of config items; `pagination: { page, page_size, total_records, count_mode }` (`count_mode` ∈ `exact` \| `auto`).
- **Errors:** 403.
- **Example:** `{ "operation": "list", "pageSize": 50, "sortModel": [{ "field": "slug", "sort": "asc" }], "filterModel": { "items": [{ "field": "enabled", "operator": "is", "value": true }], "logicOperator": "and" } }`

### `filter`

- **Purpose:** alternate list filtered by explicit columns, one-based paging.
- **Side effects:** none.
- **Inputs (all optional):** column filters `id`, `version`, `enabled`, `name`, `description`, `slug`, `fullPath`, `depth`, `createdBy`, `modifiedBy`, `parentItemId`, `isDeleted`; paging `page` (≥1, default 1) and `limit` (≤100, default 25). The endpoint's required `itemType=domain` is not a tool property (the tool is fixed to `domain`).
- **Output (200):** `data` = array of matching config items (no `pagination` block documented).
- **Errors:** 403.
- **Example:** `{ "operation": "filter", "isDeleted": false, "slug": "products" }`

### `publish`

- **Purpose:** publish the latest **saved** configuration for the slug — a multi-step publication, **publish → prune → archive**.
- **Side effects:** ⚠️ builds or refreshes live data structures. Confirm slug and intent; warn about the `track_versions` / `xref_enabled` ratchet.
- **Inputs:** `slug` (required). No body.
- **Output (200):** `data: { response, pub_id, slug, statuses: { publish, prune, archive }, step_responses: { publish, prune, archive }, has_warnings }`. Check every `statuses.*`; when `has_warnings` is `true` read `step_responses` and report the warnings. Keep `pub_id` for `get_publication`.
- **Errors → next action:**
  - 400 `VALIDATION` → the saved configuration lacks `slug` and/or `cfg_data` (not a publishable domain); fix via `update`.
  - 403 → lacks `domain.publish`.
  - 5xx `CFG_NOT_FOUND` (configuration not found — arrives as a server error, not 404) → nothing is saved under that slug; check with `get_by_slug` / `list`. Read `message` to tell it from a genuine fault.
- **Example:** `{ "operation": "publish", "slug": "products" }` → `data.statuses: { "publish": "success", "prune": "success", "archive": "success" }`, `data.has_warnings: false`

### `list_publications` / `list_published` / `get_publication` / `publications_for_domain`

- **Purpose:** publication history and status. `list_publications` = all runs (who / when / status / messages); `list_published` = successfully published domains; `get_publication` = one run by `id` (the run id, e.g. a `pub_id`); `publications_for_domain` = runs for one `domain`.
- **Side effects:** none.
- **Inputs:** none for the two lists; `id` for `get_publication`; `domain` for `publications_for_domain`.
- **Output (200):** `data` = array of records (or one run). Record fields are not documented — read them.
- **Errors:** 403; `get_publication` 404 when the run does not exist.
- **Example:** `{ "operation": "publications_for_domain", "domain": "products" }`

### `metadata_by_version` / `list_metadata`

- **Purpose:** cached per-domain metadata (columns, types, flags, masks, tenants) and per-system metadata — what is actually live.
- **Side effects:** none.
- **Inputs:** `metadata_by_version` → `domain`, `version` (positive integer; the doc does not state how the domain version relates to the config-item `version`, so confirm before assuming). `list_metadata` → optional `page` (≥0), `pageSize` (1–100), `search` (substring on slug); omit both `page` and `pageSize` for the full list.
- **Output (200):** `metadata_by_version` → `data: { domainInfoCache, domainSystemInfoCache }`; `list_metadata` → `data: { domainInfoList: [...] }` plus `pagination`.
- **Errors:** 403.
- **Example:** `{ "operation": "metadata_by_version", "domain": "products", "version": 3 }`

### `delete_published`

- **Purpose:** permanently delete a published domain and all of its data structures and metadata.
- **Side effects:** ⚠️⚠️ irreversible. Two explicit confirmations, the second repeating the slug.
- **Inputs:** `domain` (required).
- **Output (200):** `data`, `meta`, `message`.
- **Errors (stable codes) → next action:** 404 `DOMAIN_NOT_FOUND` → check `list_published`; 409 `PUBLISH_IN_FLIGHT` → wait, re-check `publications_for_domain`, retry only with the user's OK; 403 `PERMISSION_DENIED` → stop; 400 `FK_DEPENDENCY_EXISTS` → another domain references this one and must change first (finding it is outside this skill — data-model endpoints are excluded, §13); 400 `MISSING_USER_ID` / `MISSING_DOMAIN` → caller identity or slug missing.
- **Example:** `{ "operation": "delete_published", "domain": "products" }`

## Shared mechanics

### Response envelope (doc §4)

Success: `{ data, meta?, status, error: false, message?, pagination? }`. Error: `{ data: null, meta: {}, status, error: true, message, details? }`. `pagination` appears only on paginated endpoints; `message` and `meta` may be omitted or empty. Branch on `error`, then on `message` / `details`. Validation errors are always at the **top-level `details`**, never `data.details` (`data` is `null` on errors).

### Two spelling layers — never mix them

Request bodies and the tool's authoring properties are snake_case: `cfg_data`, `item_type`, `full_path`, `parent_item_id`, `domain_table_name`, `default_conflict_resolution`, `triggers_version_change`. Filter query columns and GET responses are camelCase: `fullPath`, `parentItemId`, `isDeleted`, `createdBy`, `modifiedBy`; response `itemType`, `cfgData`, `domainInfoCache`, `domainSystemInfoCache`, `domainInfoList`. The envelope's `pagination` (`page_size`, `total_records`, `count_mode`) and the publish payload (`pub_id`, `step_responses`, `has_warnings`) are snake_case. Copying `cfgData` out of a GET response into an update means sending it as `cfg_data`. Inside `cfg_data` everything is snake_case (`read_view_slug`, `record_label_attribute`, `include_cols`).

### Three paging grammars

| operation | keys | base | size | extras |
| --- | --- | --- | --- | --- |
| `list` | `page`, `pageSize` | zero-based, default 0 | 1–100, default 25 | `fetchType`, `slugContains`, `search`, `sortModel`, `filterModel` |
| `filter` | `page`, `limit` | one-based, default 1 | ≤100, default 25 | optional column filters |
| `list_metadata` | `page`, `pageSize` | ≥0 | 1–100 | `search`; omit both keys → full list |

`filterModel.items[].operator` ∈ `contains`, `doesNotContain`, `equals`, `doesNotEqual`, `startsWith`, `endsWith`, `isEmpty`, `isNotEmpty`, `is`, `not`, `isAnyOf`, `=`, `!=`, `>`, `>=`, `<`, `<=`, `after`, `before`, `onOrAfter`, `onOrBefore`. In REST, `sortModel` and `filterModel` travel as JSON-encoded query strings (§5); the tool schema types them as a real array / object — pass the structured form.

### `cfg_data` — the domain model (doc §6)

The deep validator runs at **save** (`create` / `update`), not again at `publish`. `!` = required, `?` = optional.

**Top level**

| field | type | req | notes |
| --- | --- | --- | --- |
| `domain_name_singular` | string | ! | label for a single record |
| `domain_table_name` | string | ! | **must equal the config-item `slug` exactly** (so also ≤50) |
| `default_conflict_resolution` | object | ! | `code` ! ∈ `lww` \| `fww`; `human_readable` ? (not validated) |
| `schema` | array | ! | non-empty; one entry per column (below) |
| `archive` | boolean | ? | keep historical record versions |
| `archive_policy` | object | ! when `archive=true` | `max_record_versions` ?, `retention_days` ? — non-negative integers if present; `{}` passes |
| `xref_enabled` | boolean | ? | if `true`, `track_versions` must be `true` |
| `track_versions` | boolean | ? | version records on change |
| `multi_tenant` | boolean | ? | |
| `dedupe_enabled` | boolean | ? | |
| `contact` | object | ? | not validated; `name`, `email`, `phone` free-form |
| `area` | string \| array of strings | ? | |
| `read_view_slug` | string | ? | must match `^[a-z][a-z0-9_-]{0,127}$`; existence not checked |
| `indexes` | array | ? | entries below |
| `system_access` | array | ? | top-level per-system list, entries below; **normalized on save** to only `{ system_slug, change_tracking }` per entry — other keys are dropped |
| `access_control` | object | — | server-injected on every save; never send |

**`schema[]` entry (one per column)**

| field | type | req | notes |
| --- | --- | --- | --- |
| `name` | string | ! | not reserved; no `sys_` / `_` / `p_` prefix; unique |
| `column_name` | string | ! | same rules |
| `data_type` | string | ! | see the `data_type` note below |
| `triggers_version_change` | boolean | ! | a change to this field creates a new record version |
| `description` | string | ? | |
| `indexed`, `unique`, `nullable` | boolean | ? | not validated at config layer |
| `masking` | object | ? | `masked` boolean ! when `masking` present; `mask_format` string \| null ? |
| `conflict_resolution` | object | ? | per-field override; `code` ? but if present `lww` \| `fww` |
| `fk_config` | object | ? | open object; `target_domain` ?, `alias` ? (unique across `fk_config` entries, else `DUPLICATE_ALIAS`) |
| `system_access` | array | ? | per-field overrides `{ system_slug, read_access, write_access, unmask }` — not validated at save, consumed by the access-control layer |
| `default_value` | object | ? | exactly `{ source, value }`, no other keys. `source` ! ∈ `literal` \| `refkey`. `value` !: any JSON (incl. `null`) for `literal`; a nonblank Ref Key Generator slug for `refkey`, and then `data_type` must be `text` |
| `picklist` | object | ? | `slug` ! (nonblank, `^[a-z][a-z0-9_-]{0,127}$`, existence not checked); `record_label_attribute` ? (`^[A-Za-z_][A-Za-z0-9_]*(\.[A-Za-z_][A-Za-z0-9_]*)*$`); `parameters` ? — array of `{ name !, value ! }`, `name` matching `^p_[a-z][a-z0-9_]*$`, `value` a nonblank literal or `${...}` expression; omit `parameters` to inherit the picklist's defaults. The flat `picklist_slug` / `picklist_parameters` / `picklist_source_type` keys are rejected |
| `json_doc` | marker | ? | only valid when `data_type` = `jsonb` |

**`indexes[]` entry:** `columns` ! (non-empty array; each a `schema[].column_name` or a platform meta-index column); `name` ? (≤50, `^[a-z0-9](?:[a-z0-9_]*[a-z0-9])?$`); `unique` ? (a single-column unique index duplicating an attribute-level `unique` gives a warning, not an error).

**Top-level `system_access[]` entry:** `system_slug` ! (non-empty); `change_tracking` ? — when present, `mode` ! = `specific_columns` (the only value), `include_cols` ? and `exclude_cols` ? (each entry a `schema[].column_name`).

Example column using the newer fields:

```json
{
  "name": "category", "column_name": "category", "data_type": "text", "triggers_version_change": true,
  "default_value": { "source": "literal", "value": "general" },
  "picklist": { "slug": "product_categories", "parameters": [{ "name": "p_region", "value": "emea" }] },
  "fk_config": { "target_domain": "suppliers", "alias": "supplier" }
}
```

**Reserved column names** (`name` / `column_name` must not equal any): `id`, `version`, `subversion`, `domain`, `superseded_by_id`, `is_deleted`, `deleted_at`, `version_token`, `last_tx_id`, `created_at`, `modified_at`, `created_by`, `modified_by`, `created_by_system`, `modified_by_system`, `tenant_id`, `core_record_id`, `core`, `source_record_ref`, `source_record_ct`, `has_attachment`, `updated_at`, `modified_by_source_record_ref`. None may start with `sys_`, `_` or `p_`.

Pre-save checklist — each line is a documented rejection cause:

- `slug` is 1–50 chars, matches `^[a-z0-9](?:[a-z0-9_]*[a-z0-9])?$` (no dashes, no leading/trailing `_`), and does not start with `sys_`, `dvw_`, `dtm_`, `dtv_`.
- `domain_name_singular`, `domain_table_name`, `default_conflict_resolution.code` (`lww` or `fww`) and a non-empty `schema` are present; `domain_table_name` equals `slug` exactly.
- Every `schema[]` entry has `name`, `column_name`, `data_type`, `triggers_version_change`; no name is reserved or `sys_` / `_` / `p_`-prefixed; names are unique.
- `archive: true` ⇒ `archive_policy` object present.
- `xref_enabled: true` ⇒ `track_versions: true`.
- `masking` present ⇒ `masking.masked` boolean; `default_value` is exactly `{ source, value }`, `refkey` only on `text`; `picklist.slug` present; `json_doc` only on `jsonb`; `fk_config.alias` values unique.
- `indexes[].columns` and `change_tracking.include_cols` / `exclude_cols` name existing `schema[].column_name` values.
- No `access_control`, no `domain_name`, no flat picklist keys.

**`data_type` — the two layers disagree.** The tool schema enum (previous snapshot) accepts exactly: `text`, `numeric`, `boolean`, `timestamptz`, `date`, `uuid`, `integer`, `bigint`, `jsonb`. The platform's accepted set (doc §7) is: `text`, `text[]`, `smallint`, `integer`, `bigint`, `serial`, `numeric`, `decimal`, `real`, `double precision`, `money`, `boolean`, `date`, `time`, `time with time zone`, `timestamp`, `timestamp with time zone`, `interval`, `uuid`, `jsonb`, `inet`, `cidr`, `macaddr`, `macaddr8`, `point`, `line`, `lseg`, `box`, `path`, `polygon`, `circle`, `int4range`, `int8range`, `numrange`, `tsrange`, `tstzrange`, `daterange`. Safe intersection: `text`, `numeric`, `boolean`, `date`, `uuid`, `integer`, `bigint`, `jsonb`. `timestamptz` passes the tool but is absent from the doc's list (the doc spells that type `timestamp with time zone`) — expect a possible `INVALID` at save and say so. Types outside the schema enum cannot be sent through this tool at all.

Other constants: conflict-resolution `code` ∈ `lww` (last write wins) | `fww` (first write wins); `change_tracking.mode` = `specific_columns`; `default_value.source` ∈ `literal` | `refkey`; sort ∈ `asc` | `desc`; `count_mode` ∈ `exact` | `auto`.

### Validation detail codes (inside `INVALID`)

Each top-level `details[]` entry is `{ path, code, detail, severity, remediation, proposed?, current? }`. `severity: warning` entries (e.g. `DUPLICATE_ATTR_UNIQUE`) do not fail the save — report them, don't block on them. Codes by area:

- Envelope / field shape: `REQUIRED`, `TYPE`, `VALUE`, `SLUG_TABLE_MATCH`, `RESERVED_PREFIX_SLUG`, `RESERVED_PREFIX_USDF`, `INVALID_READ_VIEW_SLUG`.
- Schema attribute: `RESERVED_ATTR_NAME`, `RESERVED_COL_NAME`, `RESERVED_PREFIX_SYS`, `RESERVED_PREFIX_UNDERSCORE`, `RESERVED_PREFIX_P`, `TRIGGERS_VERSION_CHANGE_REQUIRED`, `UNIQUE_ATTR_NAMES`, `DUPLICATE_ALIAS`, `JSON_DOC_REQUIRES_JSONB`.
- Default value / picklist: `INVALID_DEFAULT_VALUE_SHAPE`, `INVALID_DEFAULT_SOURCE`, `INVALID_DEFAULT_REFKEY_SLUG`, `INVALID_DEFAULT_REFKEY_TARGET_TYPE`, `LEGACY_FLAT_PICKLIST_TRIO`, `INVALID_PICKLIST_BINDING_SHAPE`, `PICKLIST_SLUG_REQUIRED`, `INVALID_RECORD_LABEL_ATTRIBUTE`, `INVALID_PICKLIST_PARAMETER_SHAPE`.
- Archive / cross-field: `ARCHIVE_POLICY_REQUIRED`, `XREF_REQUIRES_TRACK`.
- Indexes: `INDEX_INVALID_COLUMN`, `INDEX_DUPLICATE_DEFINITION`, `DUPLICATE_ATTR_UNIQUE` (warning).
- Change tracking: `CHANGE_TRACKING_INVALID_COLUMN`.
- Immutability & type change (update of a published domain): `IMMUTABLE_SLUG`, `IMMUTABLE_TRACK_VERSIONS_TRUE`, `IMMUTABLE_XREF_ENABLED_TRUE`, `IMMUTABLE_MULTI_TENANT_FALSE`, `IMMUTABLE_DEDUPE_TRUE`, `TYPE_CHANGE_NOT_ALLOWED`, `IMMUTABLE_DATA_TYPE`. The doc describes only the slug and `track_versions` / `xref_enabled` rules in prose; for the others, read the entry's `detail` / `remediation` / `current` and tell the user what is locked rather than retrying.

## Workflows

### A. Create and publish a new domain

1. `pivotly_ping` (once per session).
2. `get_by_slug` with the intended slug — 404 means it is free; 200 means use Workflow B instead. Check the slug against the regex first (lowercase, digits, underscores, no dashes, ≤50).
3. Build `cfg_data` from the user's columns and run the pre-save checklist; set `domain_table_name` = slug. Convention of this skill (not a system default): ask the user which columns should trigger a new record version rather than assuming `triggers_version_change`.
4. ⚠️ Show the payload; on yes → `create`. Keep `data.id`.
5. On `INVALID`, walk top-level `details[]` (`path`, `code`, `detail`, `remediation`, `proposed`), fix, resend — reconfirm if the fix changes intent.
6. ⚠️ Confirm publish (mention the ratchet when `track_versions` / `xref_enabled` is `true`) → `publish` with `slug`.
7. Read `statuses.publish` / `prune` / `archive` and `has_warnings`; report warnings from `step_responses`. Optionally `metadata_by_version` to show what went live.

### B. Change an existing domain (add or alter a column, index, default, picklist, flag)

1. `get_by_slug` → `data.id`, `data.slug`, `data.cfgData`. Always fresh — never reuse an id from memory (an unknown id upserts a new record).
2. Edit a copy of `cfgData`; drop any `access_control`. Keep `domain_table_name` unchanged; never rename `slug` on a published domain.
3. Published-domain check: do not set `track_versions` or `xref_enabled` from `true` to `false`; expect `IMMUTABLE_DATA_TYPE` if you change a live column's `data_type` and `IMMUTABLE_MULTI_TENANT_FALSE` / `IMMUTABLE_DEDUPE_TRUE` if you flip those flags — warn the user before trying.
4. ⚠️ Show the diff → `update` with `id` + `slug` (current) + the **complete** `cfg_data` (+ `name` / `description` / `enabled` only when they change). The new config-item `version` comes back.
5. ⚠️ Confirm → `publish` (nothing is live until then). Check `statuses` / `has_warnings`.

### C. Status questions ("did the publish succeed?", "what's live?")

Latest runs for one domain → `publications_for_domain` (`domain`); one run → `get_publication` (`id` = run id / `pub_id`); everything published → `list_published`; all runs → `list_publications`; live columns and flags → `metadata_by_version` / `list_metadata`.

### D. Diagnose a failed save or publish

- 400 "Schema validation" on `update` → `slug` or `cfg_data` missing, or id mismatch; resend the full record.
- `error: true` with `message` "Validation failed" (`INVALID`, arrives as 5xx) → read top-level `details`, fix each `path` using `remediation`; ignore `warning` entries for pass/fail.
- `VALUE` → slug too long or has dashes / leading or trailing `_`.
- Duplicate slug on `create` → `get_by_slug` → ⚠️ `update` the existing `id`, or choose another slug.
- `IMMUTABLE_SLUG` / other `IMMUTABLE_*` on `update` → that property is locked on the published domain; explain, offer a new domain if they truly need it changed.
- `CFG_NOT_FOUND` on `publish` → nothing saved under that slug; check spelling via `list` / `get_by_slug`.
- `update` "succeeded" but the domain looks new (version 1, missing columns) → the id was wrong and the call upserted a new record; tell the user, and with their OK `delete_config_item` the stray record.
- Any other 5xx → read `message`; if it matches none of the above, treat it as a server fault and stop.

### E. Retire a domain

1. Ask which retirement is meant: stop authoring (soft-delete the config item) or drop the live domain and its data.
2. For the live domain: `publications_for_domain` (no publish in flight) and `list_published` (it exists). ⚠️⚠️ First confirmation: state that this permanently deletes all data structures and metadata. Second confirmation: the user repeats the slug.
3. `delete_published` with `domain`. On `PUBLISH_IN_FLIGHT` wait and re-check; on `FK_DEPENDENCY_EXISTS` stop and report the dependency.
4. Optionally (⚠️ confirm) `delete_config_item` with the config-item `id` so it no longer appears as an authoring record. This ordering is the skill's convention; the doc does not prescribe one.

### F. Rename a domain

Unpublished: ⚠️ `update` with the existing `id`, the new `slug` **and** a matching `cfg_data.domain_table_name`, plus the rest of the complete `cfg_data`. Published: not possible (`IMMUTABLE_SLUG`) — create a new domain; moving records is out of scope.

## User phrasing → call

| user says | call |
| --- | --- |
| "create a domain for X", "set up a new domain", "new config item for orders" | Workflow A → ⚠️ `create` |
| "add a column to X", "make field Y indexed", "change the data type of Z", "turn on versioning for X" | Workflow B → `get_by_slug`, then ⚠️ `update` |
| "add a composite index on A and B" | Workflow B → `cfg_data.indexes[]` |
| "default this field to …", "use a ref key for the number" | Workflow B → `schema[].default_value` |
| "make this a dropdown / picklist" | Workflow B → `schema[].picklist` |
| "only send changes to columns A and B to system S" | Workflow B → top-level `system_access[].change_tracking` |
| "publish X", "push the domain live", "apply the changes" | ⚠️ `publish` `{ slug }` |
| "did the publish work", "publish status", "when was X last published" | `publications_for_domain` `{ domain }` / `get_publication` `{ id }` |
| "which domains are live / published" | `list_published` |
| "list domains", "what domains do we have", "find domains with 'cust' in the slug" | `list` (+ `slugContains`, `search`, `sortModel`, `filterModel`) |
| "show me the config for X", "what columns does X have" | `get_by_slug` `{ slug }` → read `cfgData.schema` |
| "what's actually live for X", "live metadata", "which systems can see X" | `metadata_by_version` `{ domain, version }` / `list_metadata` |
| "deleted / disabled domains", "domains created by me" | `filter` (`isDeleted`, `enabled`, `createdBy`, …) |
| "delete the config", "retire the definition but keep the data" | ⚠️ `delete_config_item` `{ id }` |
| "drop the domain", "delete X and all its data", "remove the published domain" | ⚠️⚠️ Workflow E → `delete_published` `{ domain }` |
| "rename X to Y" | Workflow F |
| "why did my save fail", "validation failed", "Schema validation", "it says IMMUTABLE_SLUG" | Workflow D |
| "is Pivotly up", "can you reach the server" | `pivotly_ping` |

## Pitfalls

- **`update` upserts.** An unknown `id` creates a new config item at that id (200, version 1) — there is no 404. Always take the id from a fresh read.
- **`update` needs the full record.** `slug` and complete `cfg_data` on every call (else 400 "Schema validation"); `cfg_data` is replaced wholesale, so an edited fragment wipes everything else. Only the scalar fields patch-merge.
- **`version` is never yours to set.** Forced to 1 on create, stored + 1 on update, no optimistic-lock check — two concurrent editors silently overwrite each other; re-read right before saving.
- **Ordinary rejections arrive as HTTP 5xx.** `INVALID` ("Validation failed", top-level `details`), `VALUE` (slug), duplicate slug, `IMMUTABLE_SLUG`, item-type change, and publish `CFG_NOT_FOUND` all surface as 5xx with a precise `message`. Branch on `error` / `message` / `details`, never on status alone; do not auto-retry. Reliable 4xx: request-shape errors (400, incl. "Schema validation"), 403, 404 on `get` / `get_by_slug` / `delete_config_item` / `get_publication`, publish `VALIDATION` (400), and every `delete_published` code.
- **`details` is top-level.** On errors `data` is `null`; the per-path list is `details`, not `data.details`.
- **Slug rule is the tighter layer's:** ≤50 (the request layer allows 255, but 51–255 then fails 5xx `VALUE`), regex `^[a-z0-9](?:[a-z0-9_]*[a-z0-9])?$` — `customer-orders` is invalid, `customer_orders` is fine.
- **`domain_table_name` must equal `slug` exactly** — at create and after any rename.
- **Ratchets on published domains:** `track_versions` / `xref_enabled` cannot return to `false`; `xref_enabled` requires `track_versions`; the slug is immutable; the doc also lists `IMMUTABLE_MULTI_TENANT_FALSE`, `IMMUTABLE_DEDUPE_TRUE` and `IMMUTABLE_DATA_TYPE` without further prose — read their `detail`.
- **Server-owned fields:** `access_control` is injected and overwritten on every save; top-level `system_access` entries are stripped to `{ system_slug, change_tracking }`. Per-field `schema[].system_access` is kept but not validated.
- **Tool schema vs platform:** `data_type` enum 9 values in the tool vs 37 in the doc; `timestamptz` is tool-only. The newer `cfg_data` fields (`area`, `read_view_slug`, `indexes`, top-level `system_access[].change_tracking`, `schema[].default_value` / `picklist` / `json_doc`) come from the doc; whether the tool's `cfg_data` schema accepts them was not re-checked against the current listing — if the tool rejects one as an unknown key, report it rather than working around it.
- **Two ids, two versions:** `id` is a config-item uuid except on `get_publication` (run id); `version` is the domain version on `metadata_by_version` and a filter column on `filter`.
- **Paging base flips:** `list` / `list_metadata` are zero-based with `pageSize`; `filter` is one-based with `limit` (send ≥1 — no minimum is enforced). `fetchType: "list"` returns everything.
- **Casing flips between layers:** send `cfg_data`, read back `cfgData`; body `full_path` / `parent_item_id`, filter `fullPath` / `parentItemId`. Never feed a response spelling back into a request.
- **Publish can "succeed" with warnings:** 200 with `has_warnings: true` — always read `statuses` and `step_responses`.
- **Saving ≠ publishing.** After `create` / `update` nothing is live until `publish`; after `publish`, a later save is not live until the next `publish`.
- **Two deletes.** `delete_config_item` leaves live data structures; `delete_published` destroys them. Confirm which.
- **Not documented — kept explicit:** the tool's exact result and failure delivery (`isError` vs envelope); whether the tool injects `item_type` on update; publication-run and published-domain record fields; how the server is authorized and whether `X-Tenant-Id` is set. Read results before chaining; ask before retrying.
- **Out of scope:** record read / write inside a domain and data-model / ERD endpoints (§13); the `filter` endpoint's `itemType` is not a tool property.
