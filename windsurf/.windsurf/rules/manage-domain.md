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

---

## File: references/domain-api.md

---
config_item: domain
plane: config-item
api_version: v3
doc_revision: 2026-09-28
generator: pci-api-refdoc-generator
---

# Domain (`domain`) — REST API Reference

A consumer-facing HTTP contract for authoring, publishing and inspecting **domains** on the Pivotly platform. It is written so that a skill or MCP tool can be built from it without reading the platform source: every field states its type, its requiredness (`!` required-to-send / `?` optional), its effective constraint, the layer and stage that enforce it, and every failure states its HTTP status and where its code appears on the wire.

**Layer tags** used throughout (categories, not implementation names):

| Tag               | Meaning                                                                              | Rejects with                    |
| ----------------- | ------------------------------------------------------------------------------------ | ------------------------------- |
| `[REST]`          | Request-shape validation at the API edge, before any platform call                   | 400                             |
| `[authz]`         | Permission check (object type + action verb)                                         | 403 (non-standard body, see §3) |
| `[service]`       | API-side checks inside the handler (type/id matching, lookups)                       | 400 / 404 / 500                 |
| `[save-fn]`       | The platform config-item save step (defaulting, injection, patch-merge, hard checks) | **500** (see §11)               |
| `[DB-validator]`  | The platform domain validator that runs inside save, on the resolved record          | **500** with a `details[]` list |
| `[DB-constraint]` | Storage constraints on the stored record                                             | **500**                         |
| `[DB-publish]`    | The platform publish process (enqueue → create → prune → archive)                    | 200 / 500 (see §9.1)            |

---

## 1. Overview

A **domain** is a governed, versioned config item (`item_type = "domain"`, identified by a globally unique `slug`) whose `cfg_data` declares a **data entity**: its attributes (columns), conflict-resolution policy, history/archive/cross-reference behaviour, tenancy, per-system change tracking and indexes. Publishing a domain turns that declaration into live storage that records can be written to and read from.

The lifecycle covered by this document:

1. **Author** the item with the generic config-item lifecycle (§8): create, read, list, update, soft-delete. Saving runs the domain validator (§6) and injects platform-managed fields; saving never creates or changes storage.
2. **Publish** it (§9.1): the platform records a publication of the **latest saved** definition and runs its create, prune and archive steps, creating or altering the domain's storage.
3. **Inspect** publications (§9.2–§9.4) and the list of published domains with their runtime metadata (§10.1).

Key behaviours a caller must know up front (each is detailed at its endpoint):

- **Saving and publishing are independent.** Edits take effect in storage only after §9.1. After any save, the domain's list entry reports `is_published: false` until it is published again at the new version (§8.4).
- **Update is a full resubmission** (§8.6): the update route requires `slug`, `item_type` and a complete `cfg_data` every time, and `cfg_data` is replaced wholesale.
- **Some capabilities are one-way once published** (§6.9): `track_versions`, `xref_enabled` and `dedupe_enabled` cannot be turned off; `multi_tenant` cannot be turned on; attribute data types cannot change; the slug cannot change.
- **Soft-delete only disables** (§8.7): the item's `enabled` becomes `false`; it is not marked deleted, and its published storage is untouched.
- **Most platform-side rejections surface as HTTP 500**, not 4xx, and platform error codes are generally **not** returned as a field — only the message (and, for save validation, a `details[]` list whose entries carry codes). A publish can also return **200** while one of its steps failed; read the step results (§9.1).

## 2. Base URL & versioning

- Every endpoint in this document lives under a configurable **base path** (default `/api`) followed by the version segment **`/v3`**.
- Full URL = `<scheme>://<host>` + `BASE_PATH` (default `/api`) + `/v3` + resource prefix + sub-path.
- Resource prefixes used:
  - `/v3/config-items` — generic config-item lifecycle (§8). The item type is a path segment: `/v3/config-items/domain/...`. The list-by-filter endpoint (§8.1) takes the type as a query parameter instead.
  - `/v3/domain-publish` — publish (§9.1).
  - `/v3/domain-publications` — publication reads (§9.2–§9.4).
  - `/v3/domain-info` — published-domain metadata (§10.1).
- Examples below assume `https://pivotly.example.com/api`.

| Prefix                    | Example full URL                                                          |
| ------------------------- | ------------------------------------------------------------------------- |
| `/v3/config-items`        | `https://pivotly.example.com/api/v3/config-items/domain/by-slug/customer` |
| `/v3/domain-publish`      | `https://pivotly.example.com/api/v3/domain-publish/customer`              |
| `/v3/domain-publications` | `https://pivotly.example.com/api/v3/domain-publications/domain/customer`  |
| `/v3/domain-info`         | `https://pivotly.example.com/api/v3/domain-info?search=cust`              |

- Request bodies are JSON (`Content-Type: application/json`). An empty or whitespace-only JSON body is treated as `{}`. On every endpoint that accepts a request body (POST / PATCH / DELETE — including publish, which ignores its body), a JSON body that does not parse is rejected with **400** (standard error envelope, `message` = the JSON parser's message), and a body sent with a content type the API does not accept is rejected with **415** (standard envelope). The maximum request body size is a deployment setting (default 10 GiB); a larger body is rejected with **413** (standard envelope).

## 3. Authentication & authorization

**Headers (every endpoint in this document):**

| Header          | Req.        | Type / format      | Purpose                                                                                                                                                                                                                                          |
| --------------- | ----------- | ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `Authorization` | !           | `Bearer <JWT>`     | An OIDC-issued access token. Must be a signed token (unsigned `alg: none` tokens are rejected) accepted for this API's issuer and audience.                                                                                                      |
| `X-Tenant-Id`   | ?           | uuid               | Tenant context passed to the permission check. When present it **must be a valid uuid** — a non-uuid value makes the permission check itself fail (**500**). Whether a caller without it is permitted depends on that caller's role assignments. |
| `X-App-Slug`    | ?           | string             | App context passed to the permission check. No format enforced.                                                                                                                                                                                  |
| `Content-Type`  | ! on bodies | `application/json` | Required for POST / PATCH / DELETE requests that carry a body.                                                                                                                                                                                   |

**Authentication failures** (standard error envelope, §4):

| Condition                                                          | Status | `message`                                                                  |
| ------------------------------------------------------------------ | ------ | -------------------------------------------------------------------------- |
| No `Authorization` header, or not `Bearer <token>`                 | 401    | `Missing bearer token`                                                     |
| Token header cannot be decoded                                     | 401    | `Invalid token format`                                                     |
| Unsigned token                                                     | 401    | `Unsecured tokens are not accepted`                                        |
| Token fails signature / issuer / audience / expiry verification    | 401    | `Token validation failed`                                                  |
| Valid token, but the identity is not registered as a platform user | 401    | `Authenticated user was not found in IAM`; `meta.code` = `USER_NOT_IN_IAM` |

**Authorization.** Every endpoint in this document is authorized by **object type `domain` + an action verb** `[authz]`. The caller's roles (evaluated in the context of `X-Tenant-Id` / `X-App-Slug`) must grant that verb on `domain`. No per-record (ACL) check applies to domains.

| Endpoint                           | Action verb                                                         |
| ---------------------------------- | ------------------------------------------------------------------- |
| §8.1 List by column filter         | `list` (on the object type named by the `itemType` query parameter) |
| §8.2 Get by slug, §8.3 Get by id   | `view`                                                              |
| §8.4 Paginated list                | `list`                                                              |
| §8.5 Create                        | `create`                                                            |
| §8.6 Update                        | `change`                                                            |
| §8.7 Soft-delete                   | `delete`                                                            |
| §9.1 Publish                       | `publish`                                                           |
| §9.2, §9.3, §9.4 Publication reads | `view`                                                              |
| §10.1 Published-domain list        | `list`                                                              |

**Authorization failures return a non-standard body** — a single `error` string, **not** the §4 envelope:

| Condition                                                                    | Status | Body                                                                                                         |
| ---------------------------------------------------------------------------- | ------ | ------------------------------------------------------------------------------------------------------------ |
| The object type could not be resolved (on §8.1: `itemType` missing or empty) | 403    | `{ "error": "Authorization Failed. Incorrect config or invalid id was provided" }`                           |
| The caller's roles do not grant the verb                                     | 403    | `{ "error": "Authorization failed. You don't have access to perform this action." }`                         |
| The permission check itself fails (e.g. `X-Tenant-Id` not a uuid)            | 500    | standard envelope; `message` is a generic authorization-check failure text — treat as an opaque server error |

**Check order.** Each endpoint states whether request validation `[REST]` or authorization runs first. Where validation runs first, a malformed request returns 400 even for a caller who lacks permission.

## 4. Standard response envelope

**Success** (all endpoints):

```json
{
  "data": "<endpoint-specific>",
  "meta": {},
  "status": 200,
  "error": false,
  "message": "optional string",
  "pagination": { "page": 0, "page_size": 25, "total_records": 3 }
}
```

| Field        | Type    | Always present | Notes                                                                                                                                |
| ------------ | ------- | -------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| `data`       | any     | yes            | Endpoint-specific payload.                                                                                                           |
| `meta`       | object  | yes            | `{}` unless the endpoint states otherwise.                                                                                           |
| `status`     | integer | yes            | Mirrors the HTTP status (200 or 201).                                                                                                |
| `error`      | boolean | yes            | Always `false`.                                                                                                                      |
| `message`    | string  | no             | Present only when the endpoint sets one (stated per endpoint).                                                                       |
| `pagination` | object  | no             | Present only on §8.4 and §10.1: `page` (integer, zero-based, or `null`), `page_size` (integer or `null`), `total_records` (integer). |

**Error** (all failures except the 403 authorization bodies in §3):

```json
{
  "data": null,
  "meta": {},
  "status": 400,
  "error": true,
  "message": "Slug is required",
  "details": [
    { "field": "data.slug", "message": "Slug is required", "code": "too_small" }
  ]
}
```

| Field     | Type    | Always present | Notes                                                                                                                                                                                                                                                                                                                      |
| --------- | ------- | -------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `data`    | null    | yes            | Always `null`.                                                                                                                                                                                                                                                                                                             |
| `meta`    | object  | yes            | Usually `{}`. On platform save failures it carries `{ action, id, slug, item_type }`; on publish failures, step context (§9.1). On the unregistered-user 401 it carries `code`.                                                                                                                                            |
| `status`  | integer | yes            | Mirrors the HTTP status.                                                                                                                                                                                                                                                                                                   |
| `error`   | boolean | yes            | Always `true`.                                                                                                                                                                                                                                                                                                             |
| `message` | string  | yes            | Human-readable reason. For request-shape failures it is the single issue's text, or `Schema validation error` when there are several issues. For platform failures this is the platform's message — the **only** place most platform conditions can be recognized (see §11).                                               |
| `details` | array   | no             | Present only when there is structured detail: (a) request-shape failures — `[{ field, message, code }]` where `code` is the request-validator issue code (§11.2); (b) domain save validation failures — `[{ path, code, detail, severity, remediation, current?, proposed? }]` where `code` is a domain rule code (§11.3). |

There is **no dedicated error-code field** in the envelope. Codes reach the caller only inside `details[]` entries (`details[].code`) or, for the unregistered-user 401, in `meta.code`.

## 5. Pagination, sorting & filtering

One endpoint uses the grid-style convention below: §8.4. (§8.1 uses `page`/`limit`, and §10.1 uses `page`/`pageSize`/`search`, each described at its endpoint; §9.2–§9.4 are not paginated.)

**Query parameters (§8.4)**

| Param         | Type                          | Req. | Bounds / format                                                                                                       | Default                | Notes                                                                                                                        |
| ------------- | ----------------------------- | ---- | --------------------------------------------------------------------------------------------------------------------- | ---------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| `page`        | integer (coerced from string) | ?    | ≥ 0                                                                                                                   | `0`                    | Zero-based. Non-numeric → 400.                                                                                               |
| `pageSize`    | integer (coerced)             | ?    | 1–100                                                                                                                 | `25`                   | Out of range → 400.                                                                                                          |
| `sortModel`   | string (JSON-encoded array)   | ?    | `[{ "field": string, "sort": "asc" \| "desc" }]`                                                                      | `createdAt` descending | **Invalid JSON or a non-conforming value is silently ignored** (default order applies). Unknown fields are silently skipped. |
| `filterModel` | string (JSON-encoded)         | ?    | Either an array of filter items, or `{ items: [...], logicOperator?, quickFilterValues?, quickFilterLogicOperator? }` | no filter              | **Invalid JSON or a non-conforming value is silently ignored** (no filtering). Items on unknown fields are silently skipped. |

**Filter item**: `{ "field": string!, "operator": <operator>!, "value"?: string | number | boolean | null | string[] | number[] }`.

**Operators** (all 21 values accepted by the request layer):

| Operator                                               | Meaning                                                                                                      |
| ------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------ |
| `contains`, `doesNotContain`, `startsWith`, `endsWith` | Case-insensitive substring / prefix / suffix match; `value` must be a string, otherwise the item is ignored. |
| `equals`, `is`, `=`                                    | Equal.                                                                                                       |
| `doesNotEqual`, `not`, `!=`                            | Not equal.                                                                                                   |
| `>`, `>=`, `<`, `<=`                                   | Comparison.                                                                                                  |
| `after`, `onOrAfter`, `before`, `onOrBefore`           | Comparison (`>`, `>=`, `<`, `<=`), intended for timestamps.                                                  |
| `isEmpty`, `isNotEmpty`                                | Null-or-empty / not-null-and-not-empty; `value` ignored.                                                     |
| `isAnyOf`                                              | Membership; `value` must be a non-empty array, otherwise the item is ignored.                                |

Comparison operators with `value` absent or `null` are ignored. On uuid-typed fields a value that is not a complete uuid is matched as a case-insensitive substring instead of by equality. A value whose type the field cannot accept (e.g. a non-numeric value on a numeric field) makes the query fail with **500**.

**logicOperator / quickFilterLogicOperator**: `and` (default) | `or`. `quickFilterValues`: array of strings; each term is matched case-insensitively as a substring across the endpoint's quick-filter fields (a term matches if any field matches); terms are combined with `quickFilterLogicOperator`. Blank terms are ignored.

**Sort directions**: `asc`, `desc`.

**Pagination response**: `pagination = { page, page_size, total_records }`; `total_records` counts all rows matching the filters.

## 6. The domain config item (cfg_data)

### 6.1 Item envelope fields (outside `cfg_data`)

These are sent inside `data` on create/update (§8.5, §8.6). Effective requiredness is resolved per endpoint there; this table gives each field's combined constraint.

| Field            | Type            | Combined constraint (caller must satisfy all)                                                                                                                                                                                                                                                                                                                                                                                                           | Layers                                                                                                                                                                                                                                                                                           |
| ---------------- | --------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `slug`           | string          | 1–**50** chars; must match `^[a-z0-9](?:[a-z0-9_]*[a-z0-9])?$` **and** `^[a-z][a-z0-9_-]{0,127}$` — together: starts with a lowercase letter, contains only lowercase letters, digits and `_`, does not end with `_`, **no dashes**; does not start with `sys_`, `dvw_`, `dtm_` or `dtv_`; unique across **all** config items of every type; must equal `cfg_data.domain_table_name`. Changeable only while the domain has never been published (§6.9). | `[REST]` 1–255; `[DB-validator]` ≤50 + first pattern (`VALUE`), `RESERVED_PREFIX_SLUG`, `RESERVED_PREFIX_USDF`, `SLUG_TABLE_MATCH`; `[DB-constraint]` second pattern (a slug starting with a digit passes the validator and is rejected by storage, 500); `[save-fn]` uniqueness and rename rule |
| `item_type`      | string          | Exactly `domain` (must equal the path segment). Immutable.                                                                                                                                                                                                                                                                                                                                                                                              | `[REST]` 1–100 chars; `[service]` equals path; `[save-fn]` immutable                                                                                                                                                                                                                             |
| `name`           | string          | ≤ 255 chars. Must resolve to a non-empty value: on create an absent or `""` `name` defaults to the slug (whitespace-only is stored as sent — create does not trim); on update values are trimmed and an empty or whitespace-only value is rejected.                                                                                                                                                                                                     | `[REST]` ≤255, not `null`; `[save-fn]` default; `[DB-validator]` `REQUIRED`                                                                                                                                                                                                                      |
| `description`    | string \| null  | ≤ 1000 chars. No format. On create stored as sent; on update trimmed, and `null` / `""` / whitespace-only clear it to `null`.                                                                                                                                                                                                                                                                                                                           | `[REST]`, `[save-fn]`                                                                                                                                                                                                                                                                            |
| `enabled`        | boolean \| null | On create `null`/absent → `true`. On update `null` is rejected by storage (500). Not checked by publish.                                                                                                                                                                                                                                                                                                                                                | `[REST]`; `[save-fn]` default; `[DB-constraint]` not null                                                                                                                                                                                                                                        |
| `full_path`      | string          | ≤ 500 chars; not `null`. On create absent or `""` → the slug (whitespace-only stored as sent). On update trimmed, and an empty or whitespace-only value is rejected by storage (500). No format enforced.                                                                                                                                                                                                                                               | `[REST]`; `[save-fn]`; `[DB-constraint]` not null                                                                                                                                                                                                                                                |
| `parent_item_id` | string (uuid)   | Must match `^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$` (case-insensitive). Not `null`. Existence of the referenced item is **not checked** anywhere.                                                                                                                                                                                                                                                                               | `[REST]`                                                                                                                                                                                                                                                                                         |
| `version`        | integer         | Positive integer if sent. **Ignored** — the platform sets version 1 on create and increments on every update.                                                                                                                                                                                                                                                                                                                                           | `[REST]`                                                                                                                                                                                                                                                                                         |
| `id`             | string (uuid)   | Uuid format. Forbidden on create; on update must equal the path id.                                                                                                                                                                                                                                                                                                                                                                                     | `[REST]`, `[service]`                                                                                                                                                                                                                                                                            |
| `cfg_data`       | object          | Required JSON object (not an array). Content rules in §6.2–§6.9.                                                                                                                                                                                                                                                                                                                                                                                        | `[REST]` object; `[DB-validator]` object (`TYPE`)                                                                                                                                                                                                                                                |

Unknown keys inside `data` (for example `is_deleted`) are **stripped** by the request layer and never reach the platform.

### 6.2 `cfg_data` — top-level fields

`cfg_data` is stored as sent, except for the two platform injections in §6.3. `[DB-validator]` rules run at **save** (create, update, soft-delete). All violations are reported together. Any key not listed here is accepted and stored without validation.

| Field                         | Type                      | Req.                               | Default                                                                                                                     | Rule (exact)                                                                                                                                                                                                                                                            | Enforced at                                              | Description                                                  |
| ----------------------------- | ------------------------- | ---------------------------------- | --------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------- | ------------------------------------------------------------ |
| `domain_name_singular`        | string                    | !                                  | —                                                                                                                           | Present and not `""` (`REQUIRED`); must be a JSON string (`TYPE`). No length or format enforced.                                                                                                                                                                        | save `[DB-validator]`                                    | Singular display name.                                       |
| `domain_table_name`           | string                    | !                                  | —                                                                                                                           | Present, not `""`, and **exactly equal to the item `slug`** (`SLUG_TABLE_MATCH`, reported for both the missing and the mismatch case).                                                                                                                                  | save `[DB-validator]`                                    | Storage name; always the slug.                               |
| `default_conflict_resolution` | object                    | !                                  | —                                                                                                                           | Present (`REQUIRED`); an object (`TYPE`); sub-fields §6.4.                                                                                                                                                                                                              | save `[DB-validator]`                                    | Domain-wide write-conflict policy.                           |
| `schema`                      | array of object           | !                                  | —                                                                                                                           | Present (`REQUIRED`); an array (`TYPE`); **at least one entry** (`VALUE`); entries per §6.5; collection rules §6.6.                                                                                                                                                     | save `[DB-validator]`                                    | Attribute (column) definitions, in column order.             |
| `archive`                     | boolean                   | ?                                  | absent counts as not-`true` in `ARCHIVE_POLICY_REQUIRED`; the registry records `false`; the publication record shows `true` | When the key is present its value must be a JSON boolean — `null` included is rejected (`TYPE`). When `true`, `archive_policy` must be an object (`ARCHIVE_POLICY_REQUIRED`).                                                                                           | save `[DB-validator]`; default at publish `[DB-publish]` | Soft-delete to archive (`true`) or hard delete (`false`).    |
| `archive_policy`              | object                    | ! when `archive` is `true`; else ? | —                                                                                                                           | Object when `archive: true`; sub-fields §6.4 (validated only when it is an object).                                                                                                                                                                                     | save `[DB-validator]`                                    | Archive retention.                                           |
| `xref_enabled`                | boolean                   | ?                                  | absent counts as not-`true` in `XREF_REQUIRES_TRACK`; the registry records `false`; the publication record shows `true`     | JSON boolean when present (`TYPE`). `true` requires `track_versions: true` (`XREF_REQUIRES_TRACK`: fires when `xref_enabled` is `true` and `track_versions` is not `true`, absent included). Cannot go `true` → `false` once published (`IMMUTABLE_XREF_ENABLED_TRUE`). | save `[DB-validator]`; default at publish                | Cross-reference tracking of source-system record references. |
| `track_versions`              | boolean                   | ?                                  | absent counts as not-`true` in `XREF_REQUIRES_TRACK`; the published-domain registry records `false`                         | JSON boolean when present (`TYPE`). Cannot go `true` → `false` once published (`IMMUTABLE_TRACK_VERSIONS_TRUE`).                                                                                                                                                        | save `[DB-validator]`                                    | Row-version history.                                         |
| `multi_tenant`                | boolean                   | ?                                  | absent: no save check applies; the published-domain registry records `false`                                                | JSON boolean when present (`TYPE`). Cannot go `false` → `true` once published (`IMMUTABLE_MULTI_TENANT_FALSE`).                                                                                                                                                         | save `[DB-validator]`                                    | Tenant isolation column.                                     |
| `dedupe_enabled`              | boolean                   | ?                                  | absent: no save check applies; the published-domain registry records `false`                                                | JSON boolean when present (`TYPE`). Cannot go `true` → `false` once published (`IMMUTABLE_DEDUPE_TRUE`).                                                                                                                                                                | save `[DB-validator]`                                    | Server-side duplicate suppression on insert.                 |
| `area`                        | string \| array of string | ?                                  | —                                                                                                                           | When present: a string, or an array whose every element is a string (`TYPE`). `null` is rejected (`TYPE`). Blank strings and empty arrays are **accepted**.                                                                                                             | save `[DB-validator]`                                    | Grouping label(s).                                           |
| `read_view_slug`              | string \| null            | ?                                  | —                                                                                                                           | When present and not `null`: a non-blank string matching `^[a-z][a-z0-9_-]{0,127}$` (`INVALID_READ_VIEW_SLUG`). The named data view's **existence is not checked** at save or publish, and it is never executed there.                                                  | save `[DB-validator]`                                    | Data view whose output enriches reads of this domain.        |
| `system_access`               | array of object           | ?                                  | —                                                                                                                           | Normalized by the platform before validation (§6.3); then an array (`TYPE`), entries per §6.7.                                                                                                                                                                          | save `[save-fn]` + `[DB-validator]`                      | Per-system change-tracking declarations (not permissions).   |
| `indexes`                     | array of object           | ?                                  | —                                                                                                                           | An array when present — `null` included is rejected (`TYPE`); entries and collection rules §6.8.                                                                                                                                                                        | save `[DB-validator]`                                    | Additional indexes.                                          |
| `access_control`              | object                    | —                                  | injected                                                                                                                    | **Always overwritten** by the platform (§6.3); any value sent is discarded.                                                                                                                                                                                             | save `[save-fn]`                                         | Platform-managed. Do not send.                               |
| `contact`                     | object                    | ?                                  | —                                                                                                                           | Not validated anywhere — any value is stored. Expected `{ name?: string }` (expected; not enforced).                                                                                                                                                                    | nowhere                                                  | Contact metadata.                                            |
| `_runtime`                    | object                    | —                                  | —                                                                                                                           | Do not send. Not populated by any endpoint in this document; if sent it is stored verbatim.                                                                                                                                                                             | nowhere                                                  | Reserved.                                                    |

⚠️ Absent capability flags are interpreted inconsistently: `ARCHIVE_POLICY_REQUIRED` and `XREF_REQUIRES_TRACK` treat an absent `archive` / `xref_enabled` / `track_versions` as not-`true`; the §6.9 immutability checks **ignore** an absent value on either side (so a domain published with `multi_tenant` absent can later be saved with `multi_tenant: true` without `IMMUTABLE_MULTI_TENANT_FALSE`, although the registry recorded `false`); the published-domain registry (§10.1) records all five absent flags as `false`; and the publication record (§9) shows an absent `archive` / `xref_enabled` as `true`. Always send `archive`, `archive_policy` (when archiving), `xref_enabled`, `track_versions`, `multi_tenant` and `dedupe_enabled` explicitly.

### 6.3 Platform injections at save (applied before validation)

| Field                     | What the platform does                                                                                                                                                                                                                                                                                                                                                                      | When                                             |
| ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------ |
| `cfg_data.access_control` | Replaced with exactly `{ "model": "all_systems", "read_access": true, "update_access": true, "insert_access": true, "delete_access": true, "hard_delete_allowed": true, "direct_write_enabled": true }`, whatever was sent.                                                                                                                                                                 | Every create, update and soft-delete `[save-fn]` |
| `cfg_data.system_access`  | When it is an array: every entry that is not a JSON object is **dropped**, every object entry is reduced to its `system_slug` (stored as text) and `change_tracking` keys only (all other keys removed), and keys whose value is `null` are removed at every depth. When every entry is dropped, it becomes `[]`. When it is not an array it is left as sent (and then rejected by `TYPE`). | Every create, update and soft-delete `[save-fn]` |

The stored `cfg_data` (returned by §8.2–§8.4 and the save responses) therefore differs from the sent `cfg_data` in these two keys.

### 6.4 Nested objects: `default_conflict_resolution`, `archive_policy`

| Field                                        | Type    | Req. | Rule (exact)                                                                                                                                        | Enforced at           |
| -------------------------------------------- | ------- | ---- | --------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------- |
| `default_conflict_resolution.code`           | string  | !    | One of `lww` (last write wins), `fww` (first write wins). Absent or any other value → `VALUE` (path `$.cfg_data.default_conflict_resolution.code`). | save `[DB-validator]` |
| `default_conflict_resolution.*` (other keys) | any     | ?    | Not validated anywhere — stored.                                                                                                                    | nowhere               |
| `archive_policy.max_record_versions`         | integer | ?    | When the key is present: a JSON number that is a whole number ≥ 0 (`VALUE`). `0` means unlimited.                                                   | save `[DB-validator]` |
| `archive_policy.retention_days`              | integer | ?    | When the key is present: a JSON number that is a whole number ≥ 0 (`VALUE`). `0` means never expire.                                                | save `[DB-validator]` |
| `archive_policy.*` (other keys)              | any     | ?    | Not validated anywhere — stored.                                                                                                                    | nowhere               |

### 6.5 `schema[]` item (attribute)

Each entry must be a JSON object (`TYPE` on `$.cfg_data.schema[<n>]`; a non-object entry is otherwise skipped). Paths below are `$.cfg_data.schema[<n>].<field>` unless stated.

| Field                                                                                                                        | Type                                | Req.                           | Rule (exact)                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Enforced at            | Description                                                                                                               |
| ---------------------------------------------------------------------------------------------------------------------------- | ----------------------------------- | ------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| `name`                                                                                                                       | string                              | !                              | Present and not `""` (`REQUIRED`). Matches `^(?!_)[A-Za-z0-9_]+$` — letters (either case), digits and `_`, not starting with `_` (`VALUE`). Not one of the reserved names (§7), compared case-insensitively (`RESERVED_ATTR_NAME`). Does not start with `sys_` (case-insensitive) (`RESERVED_PREFIX_SYS`), `_` (`RESERVED_PREFIX_UNDERSCORE`) or `p_` (case-insensitive) (`RESERVED_PREFIX_P`). No length limit enforced. Unique across `schema[]` case-insensitively (§6.6). | save `[DB-validator]`  | Logical attribute name.                                                                                                   |
| `column_name`                                                                                                                | string                              | !                              | Present and not `""` (`REQUIRED`). ≤ 63 chars (`VALUE`). Matches `^(?!_)[A-Za-z0-9_]+$` (`VALUE`). Not a reserved name, case-insensitively (`RESERVED_COL_NAME`). Does not start with `sys_` (case-insensitive) (`RESERVED_PREFIX_SYS`), `_` (`RESERVED_PREFIX_UNDERSCORE`) or `p_` (case-insensitive) (`RESERVED_PREFIX_P`). Duplicate `column_name` values are **not** checked at save.                                                                                     | save `[DB-validator]`  | Storage column name.                                                                                                      |
| `data_type`                                                                                                                  | string                              | !                              | Present and not `""` (`REQUIRED`); one of the 37 data types listed in §7 (`VALUE`). Once published it cannot change (`IMMUTABLE_DATA_TYPE`, `TYPE_CHANGE_NOT_ALLOWED` — §6.9).                                                                                                                                                                                                                                                                                                | save `[DB-validator]`  | Column type.                                                                                                              |
| `triggers_version_change`                                                                                                    | boolean                             | !                              | Must be **present** and a JSON boolean (`true`/`false`); absent, `null`, `"true"`, `1`, etc. all fail (`TRIGGERS_VERSION_CHANGE_REQUIRED`, path `/cfg_data/schema/<n>/triggers_version_change` — JSON-pointer style, unlike every other path).                                                                                                                                                                                                                                | save `[DB-validator]`  | Whether a change to this attribute creates a new row version.                                                             |
| `masking`                                                                                                                    | object                              | ?                              | When present: an object (`TYPE`); `masking.masked` must be present and a JSON boolean (`REQUIRED`, path `...masking.masked`).                                                                                                                                                                                                                                                                                                                                                 | save `[DB-validator]`  | Read-time masking.                                                                                                        |
| `masking.masked`                                                                                                             | boolean                             | ! when `masking` present       | See above.                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | save                   |                                                                                                                           |
| `masking.mask_format`                                                                                                        | string \| null                      | ?                              | Not validated anywhere — stored (expected string or null; not enforced).                                                                                                                                                                                                                                                                                                                                                                                                      | nowhere                | Mask pattern.                                                                                                             |
| `conflict_resolution`                                                                                                        | object                              | ?                              | When present: an object (`TYPE`).                                                                                                                                                                                                                                                                                                                                                                                                                                             | save `[DB-validator]`  | Per-attribute override of the domain default.                                                                             |
| `conflict_resolution.code`                                                                                                   | string                              | ?                              | When present, one of `lww`, `fww` (`VALUE`, path `...conflict_resolution.code`). An **absent** `code` is not rejected.                                                                                                                                                                                                                                                                                                                                                        | save `[DB-validator]`  |                                                                                                                           |
| `fk_config`                                                                                                                  | object                              | ?                              | When present: an object (`TYPE`). Its sub-fields are not validated at save, except that `alias` takes part in the `DUPLICATE_ALIAS` rule (§6.6).                                                                                                                                                                                                                                                                                                                              | save `[DB-validator]`  | Foreign-key relationship to a parent domain.                                                                              |
| `fk_config.alias`                                                                                                            | string                              | ?                              | Default = `fk_config.target_domain`. Unique across the attributes that have `fk_config` (§6.6). No format enforced.                                                                                                                                                                                                                                                                                                                                                           | save (uniqueness only) | Disambiguator for several FKs to one target.                                                                              |
| `fk_config.target_domain`                                                                                                    | string                              | ?                              | Not validated at save (expected: a domain slug; existence not checked at save).                                                                                                                                                                                                                                                                                                                                                                                               | nowhere at save        | Parent domain.                                                                                                            |
| `fk_config.target_attr`, `required`, `resolution_policy`, `source_ref_attr`, `unique_col_lookup`, `unique_col_lookup_target` | string / boolean / string           | ?                              | Not validated at save. Expected: `target_attr` string (default `id`); `required` boolean; `resolution_policy` one of `do_nothing`, `lookup_only`, `strict`, `auto_create`; the others strings (expected; not enforced at save).                                                                                                                                                                                                                                               | nowhere at save        | FK resolution settings used when records are written. ⚠️ Their write-time behaviour is outside this document's endpoints. |
| `json_doc`                                                                                                                   | object                              | ?                              | Allowed only when `data_type` is `jsonb` (`JSON_DOC_REQUIRES_JSONB`, fires when the key is present and `data_type` is present and not `jsonb`). Its structure is not validated.                                                                                                                                                                                                                                                                                               | save `[DB-validator]`  | Declared shape of a JSON attribute.                                                                                       |
| `default_value`                                                                                                              | object \| null                      | ?                              | When present and not `null`: must be an object (`INVALID_DEFAULT_VALUE_SHAPE`) whose only keys are `source` and `value` (each other key → `INVALID_DEFAULT_VALUE_SHAPE` at `...default_value.<key>`). Sub-fields below.                                                                                                                                                                                                                                                       | save `[DB-validator]`  | Value applied when the attribute is omitted on insert.                                                                    |
| `default_value.source`                                                                                                       | string                              | ! when `default_value` present | `literal` or `refkey`; absent, `""` or other → `INVALID_DEFAULT_SOURCE`.                                                                                                                                                                                                                                                                                                                                                                                                      | save                   |                                                                                                                           |
| `default_value.value`                                                                                                        | any (literal) / string (refkey)     | ! when `default_value` present | `literal`: the key must be present (`INVALID_DEFAULT_VALUE_SHAPE` if absent); any JSON value, `null` included; type compatibility is not checked at save. `refkey`: a non-blank JSON string (`INVALID_DEFAULT_REFKEY_SLUG`) — the slug of a reference-key generator; no further format is enforced and its **existence is not checked at save**; and the attribute's `data_type` must be `text` (`INVALID_DEFAULT_REFKEY_TARGET_TYPE`).                                       | save                   |                                                                                                                           |
| `picklist`                                                                                                                   | object \| null                      | ?                              | When present and not `null`: an object (`INVALID_PICKLIST_BINDING_SHAPE`). Sub-fields below. The picklist's **existence is not checked at save**.                                                                                                                                                                                                                                                                                                                             | save `[DB-validator]`  | Default picklist that constrains this attribute.                                                                          |
| `picklist.slug`                                                                                                              | string                              | ! when `picklist` present      | Non-empty (`PICKLIST_SLUG_REQUIRED`); matches `^[a-z][a-z0-9_-]{0,127}$` (`VALUE`).                                                                                                                                                                                                                                                                                                                                                                                           | save                   |                                                                                                                           |
| `picklist.record_label_attribute`                                                                                            | string                              | ?                              | When the key is present: a non-blank string matching `^[A-Za-z_][A-Za-z0-9_]*(\.[A-Za-z_][A-Za-z0-9_]*)*$` (`INVALID_RECORD_LABEL_ATTRIBUTE`). The named attribute's existence is not checked.                                                                                                                                                                                                                                                                                | save                   | Attribute (or dot path) that stores the selected option's label.                                                          |
| `picklist.parameters`                                                                                                        | array of object \| null             | ?                              | When present and not `null`: an array (`INVALID_PICKLIST_BINDING_SHAPE`). Entries are overrides only; a partial set is valid.                                                                                                                                                                                                                                                                                                                                                 | save                   | Picklist parameter overrides.                                                                                             |
| `picklist.parameters[].name`                                                                                                 | string                              | !                              | Non-empty and matching `^p_[a-z][a-z0-9_]*$` (`INVALID_PICKLIST_PARAMETER_SHAPE`). An entry that is not an object → `INVALID_PICKLIST_PARAMETER_SHAPE` at `...parameters[<m>]`.                                                                                                                                                                                                                                                                                               | save                   |                                                                                                                           |
| `picklist.parameters[].value`                                                                                                | string                              | !                              | A non-empty JSON string — a literal or a `${...}` expression resolved when the picklist runs (`INVALID_PICKLIST_PARAMETER_SHAPE`).                                                                                                                                                                                                                                                                                                                                            | save                   |                                                                                                                           |
| `picklist_slug`, `picklist_parameters`, `picklist_source_type`                                                               | —                                   | must be absent                 | Any of these keys present → `LEGACY_FLAT_PICKLIST_TRIO` (path `$.cfg_data.schema[<n>]`).                                                                                                                                                                                                                                                                                                                                                                                      | save                   | Retired flat binding keys.                                                                                                |
| `unique`                                                                                                                     | boolean                             | ?                              | Not type-checked. Read (as a boolean) only by the `DUPLICATE_ATTR_UNIQUE` warning (§6.8); a value that cannot be read as a boolean there makes validation fail with `INVALID`. Expected boolean (expected; enforced only as described).                                                                                                                                                                                                                                       | save (indirectly)      | Per-attribute unique constraint.                                                                                          |
| `description`, `nullable`, `indexed`, `notes`                                                                                | string / boolean / boolean / string | ?                              | Not validated at save — stored (expected types as shown; not enforced).                                                                                                                                                                                                                                                                                                                                                                                                       | nowhere at save        | ⚠️ Their effect is applied at publish; not determinable here.                                                             |

### 6.6 `schema[]` collection rules

| Rule (exact predicate)                                                                                                                                                                                                         | Code                | Path                | Remediation        |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------- | ------------------- | ------------------ |
| Two or more object entries have the same non-empty `name`, compared **case-insensitively** (one entry per duplicated name, `current` = the lower-cased name)                                                                   | `UNIQUE_ATTR_NAMES` | `$.cfg_data.schema` | `rename_and_retry` |
| Two or more entries that have an `fk_config` key share the same alias, where alias = `fk_config.alias` if non-empty, else `fk_config.target_domain` if non-empty (compared case-sensitively; entries with neither are ignored) | `DUPLICATE_ALIAS`   | `$.cfg_data.schema` | `rename_and_retry` |

### 6.7 `system_access[]` item

Validated **after** the §6.3 normalization (so non-object entries have already been removed and only `system_slug` / `change_tracking` remain).

| Field                          | Type            | Req.                             | Rule (exact)                                                                                                                                                                                                                                                 | Enforced at           |
| ------------------------------ | --------------- | -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | --------------------- |
| entry                          | object          | —                                | Must be an object (`TYPE` at `$.cfg_data.system_access[<n>]` — cannot occur after normalization).                                                                                                                                                            | save                  |
| `system_slug`                  | string          | !                                | Present and non-empty (`REQUIRED`). **No format is enforced** and the named system's existence is **not checked** at save. The §6.3 normalization stores it as text, so a non-string value (number, object) is converted to its string form and then passes. | save `[DB-validator]` |
| `change_tracking`              | object          | ?                                | When present: an object (`TYPE`).                                                                                                                                                                                                                            | save `[DB-validator]` |
| `change_tracking.mode`         | string          | ! when `change_tracking` present | Present and non-empty (`REQUIRED`); must be exactly `specific_columns` (`VALUE`).                                                                                                                                                                            | save `[DB-validator]` |
| `change_tracking.include_cols` | array of string | ?                                | When it is an array: every element must equal (case-sensitively) the `column_name` of some `schema[]` entry; a `null` element also fails (`CHANGE_TRACKING_INVALID_COLUMN`, path `...change_tracking.include_cols[<m>]`). A non-array value is ignored.      | save `[DB-validator]` |
| `change_tracking.exclude_cols` | array of string | ?                                | Same rule as `include_cols` (`CHANGE_TRACKING_INVALID_COLUMN`, path `...exclude_cols[<m>]`).                                                                                                                                                                 | save `[DB-validator]` |
| any other key                  | —               | —                                | Removed by the §6.3 normalization.                                                                                                                                                                                                                           | save `[save-fn]`      |

### 6.8 `indexes[]` item and collection rules

| Field         | Type            | Req. | Rule (exact)                                                                                                                                                                                                                                                                                                                                   | Enforced at           |
| ------------- | --------------- | ---- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------- |
| entry         | object          | —    | Must be an object (`TYPE` at `$.cfg_data.indexes[<n>]`).                                                                                                                                                                                                                                                                                       | save                  |
| `columns`     | array of string | !    | A non-empty array (`REQUIRED` at `...indexes[<n>].columns`). Each non-null element must be the `column_name` of a `schema[]` entry **or** one of the meta index columns `id`, `tenant_id`, `is_deleted`, `created_at`, `modified_at`, `version`, `version_token`, `last_tx_id` (case-sensitive) (`INDEX_INVALID_COLUMN` at `...columns[<m>]`). | save `[DB-validator]` |
| `name`        | string          | ?    | When non-empty: ≤ 50 chars (`VALUE`) and matches `^[a-z0-9](?:[a-z0-9_]*[a-z0-9])?$` (`VALUE`).                                                                                                                                                                                                                                                | save `[DB-validator]` |
| `unique`      | boolean         | ?    | Not required and not type-checked at save. Read as a boolean only for single-column indexes (for the warning below); a value that cannot be read as a boolean there makes validation fail with `INVALID`. Expected boolean (expected; enforced only as described).                                                                             | save (indirectly)     |
| any other key | any             | ?    | Not validated — stored.                                                                                                                                                                                                                                                                                                                        | nowhere               |

| Collection rule (exact predicate)                                                                           | Code                                                                                   | Severity                                                      |
| ----------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------- | ------------------------------------------------------------- |
| An index's `columns` array is identical (same elements, same order) to that of an earlier index in the list | `INDEX_DUPLICATE_DEFINITION` (path `$.cfg_data.indexes[<n>]`, `current` = the columns) | error                                                         |
| A single-column index with `unique: true` names a column whose `schema[]` entry also has `unique: true`     | `DUPLICATE_ATTR_UNIQUE`                                                                | **warning** — never returned to the caller; the save succeeds |

### 6.9 Immutability and rename rules

"Published" below means the domain has an entry in the platform's published-domain registry (the list returned by §10.1).

| Rule (exact)                                                                                                                                                                                                                                                                                                                                                                                                | Code / result                                                                                                       | Stage                 |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- | --------------------- |
| Changing `slug` on update when the **current** slug is published                                                                                                                                                                                                                                                                                                                                            | 500, message states that the slug rename is blocked because the domain is published and ends `Code: IMMUTABLE_SLUG` | save `[save-fn]`      |
| Changing `slug` on update when the current slug is **not** published                                                                                                                                                                                                                                                                                                                                        | Allowed (the new slug must satisfy every §6.1 rule, be unused, and `domain_table_name` must be changed to match).   | —                     |
| `track_versions` was `true` in the published configuration and is now `false` (absent does not count)                                                                                                                                                                                                                                                                                                       | `IMMUTABLE_TRACK_VERSIONS_TRUE`                                                                                     | save `[DB-validator]` |
| `xref_enabled` was `true` and is now `false`                                                                                                                                                                                                                                                                                                                                                                | `IMMUTABLE_XREF_ENABLED_TRUE`                                                                                       | save `[DB-validator]` |
| `multi_tenant` was `false` and is now `true`                                                                                                                                                                                                                                                                                                                                                                | `IMMUTABLE_MULTI_TENANT_FALSE`                                                                                      | save `[DB-validator]` |
| `dedupe_enabled` was `true` and is now `false`                                                                                                                                                                                                                                                                                                                                                              | `IMMUTABLE_DEDUPE_TRUE`                                                                                             | save `[DB-validator]` |
| For an attribute whose `column_name` exists in the published configuration, `data_type` differs from the published value (exact string comparison)                                                                                                                                                                                                                                                          | `IMMUTABLE_DATA_TYPE` (path `...schema[<n>].data_type`, `current` = published, `proposed` = sent)                   | save `[DB-validator]` |
| The domain's storage exists and, for an attribute whose `column_name` is an existing column, the column's actual type differs from `data_type` after type-name normalization (e.g. `int4` = `integer`). ⚠️ `serial` never normalizes to a column type, so once storage exists **every** save of a domain with a `serial` attribute raises this code (current `integer`, proposed `serial`) — avoid `serial` | `TYPE_CHANGE_NOT_ALLOWED` (path `...schema[<n>].data_type`)                                                         | save `[DB-validator]` |

Adding attributes, removing attributes, enabling `track_versions` / `xref_enabled` / `dedupe_enabled`, and disabling `multi_tenant` are not blocked at save.

## 7. Constants & enumerations

| Set                                                                                      | Values (complete)                                                                                                                                                                                                                                                                                                                                                                                                             |
| ---------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `item_type` for this document                                                            | `domain`                                                                                                                                                                                                                                                                                                                                                                                                                      |
| Config item types with a save-time validator                                             | `domain`, `template`, `picklist`, `form`, `action`, `rule_set`, `process`, `event_rule`, `data_view`, `data_extension`, `app`, `app_page`, `data_model`, `agent`, `grounding_recipe`, `harvester`, `term_set`, `criterion`, `criteria_pack`, `content`, `context`, `knowledge_analysis`, `answer_view_contract`, `reference_set`, `corpus`                                                                                    |
| Domain slug (combined)                                                                   | `^[a-z](?:[a-z0-9_]*[a-z0-9])?$`, max 50 chars                                                                                                                                                                                                                                                                                                                                                                                |
| Reserved slug prefixes (rejected)                                                        | `sys_` (`RESERVED_PREFIX_SLUG`); `dvw_`, `dtm_`, `dtv_` (`RESERVED_PREFIX_USDF`)                                                                                                                                                                                                                                                                                                                                              |
| Attribute `name` / `column_name` pattern                                                 | `^(?!_)[A-Za-z0-9_]+$` (`column_name` also ≤ 63 chars)                                                                                                                                                                                                                                                                                                                                                                        |
| Reserved attribute prefixes (rejected)                                                   | `_`, `sys_`, `p_` (the last two case-insensitive)                                                                                                                                                                                                                                                                                                                                                                             |
| Reserved attribute / column names (case-insensitive)                                     | `id`, `version`, `subversion`, `domain`, `superseded_by_id`, `is_deleted`, `deleted_at`, `version_token`, `last_tx_id`, `created_at`, `modified_at`, `created_by`, `modified_by`, `created_by_system`, `modified_by_system`, `tenant_id`, `core_record_id`, `core`, `source_record_ref`, `source_record_ct`, `has_attachment`, `updated_at`, `modified_by_source_record_ref`                                                  |
| Meta index columns (allowed in `indexes[].columns` without a `schema[]` entry)           | `id`, `tenant_id`, `is_deleted`, `created_at`, `modified_at`, `version`, `version_token`, `last_tx_id`                                                                                                                                                                                                                                                                                                                        |
| Attribute `data_type` (37 values)                                                        | `text`, `text[]`, `smallint`, `integer`, `bigint`, `serial`, `numeric`, `decimal`, `real`, `double precision`, `money`, `boolean`, `date`, `time`, `time with time zone`, `timestamp`, `timestamp with time zone`, `interval`, `uuid`, `jsonb`, `inet`, `cidr`, `macaddr`, `macaddr8`, `point`, `line`, `lseg`, `box`, `path`, `polygon`, `circle`, `int4range`, `int8range`, `numrange`, `tsrange`, `tstzrange`, `daterange` |
| Conflict-resolution `code`                                                               | `lww`, `fww`                                                                                                                                                                                                                                                                                                                                                                                                                  |
| `change_tracking.mode`                                                                   | `specific_columns`                                                                                                                                                                                                                                                                                                                                                                                                            |
| `default_value.source`                                                                   | `literal`, `refkey`                                                                                                                                                                                                                                                                                                                                                                                                           |
| Index `name` pattern                                                                     | `^[a-z0-9](?:[a-z0-9_]*[a-z0-9])?$`, max 50 chars                                                                                                                                                                                                                                                                                                                                                                             |
| Picklist binding slug / `read_view_slug` pattern                                         | `^[a-z][a-z0-9_-]{0,127}$`                                                                                                                                                                                                                                                                                                                                                                                                    |
| Picklist parameter name pattern                                                          | `^p_[a-z][a-z0-9_]*$`                                                                                                                                                                                                                                                                                                                                                                                                         |
| `record_label_attribute` pattern                                                         | `^[A-Za-z_][A-Za-z0-9_]*(\.[A-Za-z_][A-Za-z0-9_]*)*$`                                                                                                                                                                                                                                                                                                                                                                         |
| Injected `access_control` (fixed)                                                        | `model: "all_systems"`, `read_access: true`, `update_access: true`, `insert_access: true`, `delete_access: true`, `hard_delete_allowed: true`, `direct_write_enabled: true`                                                                                                                                                                                                                                                   |
| Publication `status`                                                                     | `not_set`, `staged`, `preprocessing`, `pending`, `processing`, `complete`, `cancel_requested`                                                                                                                                                                                                                                                                                                                                 |
| Publication `publishPhase`                                                               | `not_set`, `not_started`, `pending_create`, `create`, `pending_prune`, `prune`, `pending_archive_update`, `archive_update`, `complete`                                                                                                                                                                                                                                                                                        |
| Step results (`createResult`, `pruneResult`, `archiveResult`; publish `response.result`) | `not_set`, `success`, `warning`, `error`, `skipped`, `cleared`, `canceled`, `not_found`                                                                                                                                                                                                                                                                                                                                       |
| Save-response `meta.action`                                                              | `insert`, `update`                                                                                                                                                                                                                                                                                                                                                                                                            |
| Validation detail `severity`                                                             | `error` (warnings are never returned)                                                                                                                                                                                                                                                                                                                                                                                         |
| Validation detail `remediation`                                                          | `none`, `rename_and_retry`, `drop_and_recreate`                                                                                                                                                                                                                                                                                                                                                                               |
| Action verbs (platform-wide)                                                             | `*`, `create`, `view`, `list`, `change`, `delete`, `manage`, `download`, `import`, `export`, `publish`, `sync`, `run`, `terminate`, `cancel`, `retry`, `pause`, `resume`, `clean`, `clear`, `resolve`, `chat` — this document uses `list`, `view`, `create`, `change`, `delete`, `publish`                                                                                                                                    |
| Sort directions                                                                          | `asc`, `desc`                                                                                                                                                                                                                                                                                                                                                                                                                 |
| Filter operators                                                                         | see §5 (21 values)                                                                                                                                                                                                                                                                                                                                                                                                            |
| Filter logic operators                                                                   | `and`, `or`                                                                                                                                                                                                                                                                                                                                                                                                                   |
| `fetchType` (§8.4)                                                                       | `list`, `paginated`                                                                                                                                                                                                                                                                                                                                                                                                           |

## 8. Endpoint reference — Authoring lifecycle

**Config item read record.** §8.1–§8.3 return config items in this shape (camelCase keys); §8.3 adds `publish_status`, and §8.4 returns a reduced shape plus `is_published` (stated there). Restated at each endpoint.

| Field                                     | Type                        | Notes                                                                                                                                                                                                        |
| ----------------------------------------- | --------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `id`                                      | string (uuid)               | Item id.                                                                                                                                                                                                     |
| `itemType`                                | string                      | `domain`.                                                                                                                                                                                                    |
| `isDeleted`                               | boolean                     | Always `false` for items managed through this API (see §8.7).                                                                                                                                                |
| `version`                                 | integer                     | Starts at 1; +1 on every update or soft-delete.                                                                                                                                                              |
| `enabled`                                 | boolean                     | `false` after §8.7.                                                                                                                                                                                          |
| `parentItemId`                            | null                        | Always `null`: the value is stored as a uuid but the read layer converts it to a number, which fails and is serialized as `null`. The stored `parent_item_id` is visible only in save responses (§8.5–§8.7). |
| `name`                                    | string                      | Display name.                                                                                                                                                                                                |
| `description`                             | string \| null              |                                                                                                                                                                                                              |
| `slug`                                    | string                      |                                                                                                                                                                                                              |
| `fullPath`                                | string                      |                                                                                                                                                                                                              |
| `depth`                                   | integer                     | Server-computed: 1 + number of `/` characters in `fullPath`.                                                                                                                                                 |
| `cfgData`                                 | object                      | The **stored** `cfg_data` — including the injected `access_control` and the normalized `system_access` (§6.3).                                                                                               |
| `createdAt`, `modifiedAt`                 | string (ISO-8601 timestamp) |                                                                                                                                                                                                              |
| `createdBy`, `modifiedBy`                 | string (uuid) \| null       | Platform user ids.                                                                                                                                                                                           |
| `createdByIdentity`, `modifiedByIdentity` | string \| null              | Display labels of those users.                                                                                                                                                                               |
| `deletedAt`                               | string \| null              | Always `null` for items managed through this API.                                                                                                                                                            |

**Save response row.** §8.5–§8.7 return the stored row in this shape (snake_case keys; timestamps are ISO-8601 with a numeric offset and up to six fractional-second digits, trailing zeros omitted, e.g. `2026-09-28T01:15:22.310412+00:00` — ⚠️ the offset shown depends on the platform's session time zone), restated at each endpoint: `id` (uuid), `item_type`, `is_deleted` (boolean), `deleted_at` (null), `version` (integer), `enabled` (boolean), `parent_item_id` (uuid \| null), `name`, `description` (string \| null), `slug`, `full_path`, `depth` (integer), `cfg_data` (object — stored form, §6.3), `created_at`, `created_by` (uuid \| null), `modified_at`, `modified_by` (uuid \| null).

### 8.1 List domains by column filter — GET /v3/config-items

- **Classification:** consumer.
- **Purpose:** exact-match lookup of config items by one or more columns, AND-combined. Returns an unpaginated page of rows, in no guaranteed order.
- **Headers:** `Authorization: Bearer <JWT>` (!), `X-Tenant-Id` (?), `X-App-Slug` (?).
- **Applicable global rules:** §2 base URL; §3 authentication, authorization (verb `list` on the object type named by `itemType`), 403 body shape; §4 envelopes.
- **Check order:** `[authz]` **first**, then `[REST]` query validation.

**Query parameters** (every value arrives as a string; conversions as stated). Text filters are **exact, case-sensitive equality**.

| Param          | Type              | Req. | Bounds / format                                                                                                                              | Enforced                                  | Notes                                                                     |
| -------------- | ----------------- | ---- | -------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------- | ------------------------------------------------------------------------- |
| `itemType`     | string            | !    | Length ≥ 1; must be `domain` for this document                                                                                               | `[authz]` (missing/empty → 403), `[REST]` | Scopes results to that type; also the object type used for authorization. |
| `id`           | string            | ?    | None at the request layer; must be a full uuid or the query fails (500)                                                                      | —                                         |                                                                           |
| `slug`         | string            | ?    | None enforced                                                                                                                                | —                                         |                                                                           |
| `name`         | string            | ?    | None enforced                                                                                                                                | —                                         |                                                                           |
| `description`  | string            | ?    | None enforced                                                                                                                                | —                                         |                                                                           |
| `fullPath`     | string            | ?    | None enforced                                                                                                                                | —                                         |                                                                           |
| `version`      | number (coerced)  | ?    | Must be numeric (non-numeric → 400)                                                                                                          | `[REST]`                                  |                                                                           |
| `depth`        | number (coerced)  | ?    | Must be numeric (non-numeric → 400)                                                                                                          | `[REST]`                                  |                                                                           |
| `enabled`      | boolean-like      | ?    | `true`/`false`/`1`/`0` (case-insensitive) → boolean; any other string is passed through and makes the query fail (500)                       | `[REST]` conversion only                  |                                                                           |
| `isDeleted`    | boolean-like      | ?    | Same as `enabled`                                                                                                                            | `[REST]` conversion only                  | Always `false` for domains (§8.7).                                        |
| `createdBy`    | string            | ?    | Must be a full uuid or the query fails (500)                                                                                                 | —                                         |                                                                           |
| `modifiedBy`   | string            | ?    | Must be a full uuid or the query fails (500)                                                                                                 | —                                         |                                                                           |
| `parentItemId` | string            | ?    | ⚠️ Likely matches a full uuid (compared against the stored uuid); a non-uuid value fails the query (500). Not fully determinable from source | —                                         |                                                                           |
| `page`         | integer (coerced) | ?    | ≥ 1                                                                                                                                          | `[REST]`                                  | Default `1` (one-based on this endpoint).                                 |
| `limit`        | integer (coerced) | ?    | ≤ 100; **no minimum** — `0` returns `[]`, a negative value fails (500)                                                                       | `[REST]`                                  | Default `25`.                                                             |

Unknown query parameters are stripped.

**Success — 200**

```json
{
  "data": [
    {
      "id": "5b0e8f3a-2c4d-4e6f-8a1b-3c5d7e9f1a2b",
      "itemType": "domain",
      "isDeleted": false,
      "version": 2,
      "enabled": true,
      "parentItemId": null,
      "name": "Customers",
      "description": "Customer master records",
      "slug": "customer",
      "fullPath": "customer",
      "depth": 1,
      "cfgData": {
        "domain_name_singular": "Customer",
        "domain_table_name": "customer",
        "default_conflict_resolution": { "code": "lww" },
        "archive": true,
        "archive_policy": { "max_record_versions": 10000, "retention_days": 0 },
        "xref_enabled": true,
        "track_versions": true,
        "multi_tenant": false,
        "dedupe_enabled": false,
        "area": "CRM",
        "schema": [
          {
            "name": "customer_name",
            "column_name": "customer_name",
            "data_type": "text",
            "nullable": false,
            "triggers_version_change": true
          },
          {
            "name": "email",
            "column_name": "email",
            "data_type": "text",
            "unique": true,
            "triggers_version_change": false
          },
          {
            "name": "status",
            "column_name": "status",
            "data_type": "text",
            "triggers_version_change": false,
            "picklist": { "slug": "pkl-customer-status" }
          }
        ],
        "indexes": [{ "columns": ["status", "created_at"], "unique": false }],
        "access_control": {
          "model": "all_systems",
          "read_access": true,
          "update_access": true,
          "insert_access": true,
          "delete_access": true,
          "hard_delete_allowed": true,
          "direct_write_enabled": true
        }
      },
      "createdAt": "2026-09-20T02:11:05.120Z",
      "createdBy": "7d2b9a10-4c3e-4f5a-9b8c-1d2e3f4a5b6c",
      "createdByIdentity": "Jordan Lee",
      "modifiedAt": "2026-09-27T08:40:12.004Z",
      "modifiedBy": "7d2b9a10-4c3e-4f5a-9b8c-1d2e3f4a5b6c",
      "modifiedByIdentity": "Jordan Lee",
      "deletedAt": null
    }
  ],
  "meta": {},
  "status": 200,
  "error": false
}
```

`data` is an array of config item read records, each with: `id` (uuid), `itemType` (`domain`), `isDeleted` (boolean, always `false`), `version` (integer), `enabled` (boolean), `parentItemId` (always `null`), `name` (string), `description` (string \| null), `slug` (string), `fullPath` (string), `depth` (integer), `cfgData` (object, stored form), `createdAt` / `modifiedAt` (ISO timestamp), `createdBy` / `modifiedBy` (uuid \| null), `createdByIdentity` / `modifiedByIdentity` (string \| null), `deletedAt` (null). No `pagination`, no `message`, no publish status.

**Errors**

| Status | Trigger                                                                                                                     | `message` / body                                                                     | Code location                          |
| ------ | --------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------ | -------------------------------------- |
| 401    | Authentication failure (§3)                                                                                                 | per §3                                                                               | `meta.code` only for `USER_NOT_IN_IAM` |
| 403    | `itemType` missing or empty                                                                                                 | `{ "error": "Authorization Failed. Incorrect config or invalid id was provided" }`   | not returned                           |
| 403    | Caller lacks `list` on `domain`                                                                                             | `{ "error": "Authorization failed. You don't have access to perform this action." }` | not returned                           |
| 400    | Query shape invalid (non-numeric `version`/`depth`/`page`/`limit`, `page` < 1, `limit` > 100)                               | single issue message, or `Schema validation error` for several                       | `details[].code` (§11.2)               |
| 500    | Query failed (non-uuid `id`/`createdBy`/`modifiedBy`/`parentItemId`, unconvertible `enabled`/`isDeleted`, negative `limit`) | `Failed to fetch config items by columns`                                            | not returned                           |

**Example**

```bash
curl -G "https://pivotly.example.com/api/v3/config-items" \
  -H "Authorization: Bearer $TOKEN" -H "X-Tenant-Id: $TENANT_ID" \
  --data-urlencode "itemType=domain" \
  --data-urlencode "slug=customer"
```

**Notes:** read-only. Use §8.4 for search, sorting, paging and publish status.

### 8.2 Get a domain by slug — GET /v3/config-items/domain/by-slug/{slug}

- **Classification:** consumer.
- **Headers:** `Authorization: Bearer <JWT>` (!), `X-Tenant-Id` (?), `X-App-Slug` (?).
- **Applicable global rules:** §2; §3 (verb `view`), 403 body shape; §4.
- **Check order:** `[REST]` params → `[authz]`.

**Path parameters**

| Param  | Type   | Req. | Bounds / format                                                                                           |
| ------ | ------ | ---- | --------------------------------------------------------------------------------------------------------- |
| `slug` | string | !    | Length ≥ 1 `[REST]`; no pattern enforced — any non-empty value is looked up; exact, case-sensitive match. |

**Success — 200**: `data` is one config item read record — `id` (uuid), `itemType` (`domain`), `isDeleted` (boolean, `false`), `version` (integer), `enabled` (boolean), `parentItemId` (always `null`), `name` (string), `description` (string \| null), `slug` (string), `fullPath` (string), `depth` (integer), `cfgData` (object, stored form), `createdAt` / `modifiedAt` (ISO timestamp), `createdBy` / `modifiedBy` (uuid \| null), `createdByIdentity` / `modifiedByIdentity` (string \| null), `deletedAt` (null). No publish status. `message` = `Config item retrieved successfully`.

```json
{
  "data": {
    "id": "5b0e8f3a-2c4d-4e6f-8a1b-3c5d7e9f1a2b",
    "itemType": "domain",
    "isDeleted": false,
    "version": 2,
    "enabled": true,
    "parentItemId": null,
    "name": "Customers",
    "description": "Customer master records",
    "slug": "customer",
    "fullPath": "customer",
    "depth": 1,
    "cfgData": {
      "domain_name_singular": "Customer",
      "domain_table_name": "customer",
      "default_conflict_resolution": { "code": "lww" },
      "archive": true,
      "archive_policy": { "max_record_versions": 10000, "retention_days": 0 },
      "xref_enabled": true,
      "track_versions": true,
      "multi_tenant": false,
      "dedupe_enabled": false,
      "area": "CRM",
      "schema": [
        {
          "name": "customer_name",
          "column_name": "customer_name",
          "data_type": "text",
          "nullable": false,
          "triggers_version_change": true
        },
        {
          "name": "email",
          "column_name": "email",
          "data_type": "text",
          "unique": true,
          "triggers_version_change": false
        },
        {
          "name": "status",
          "column_name": "status",
          "data_type": "text",
          "triggers_version_change": false,
          "picklist": { "slug": "pkl-customer-status" }
        }
      ],
      "indexes": [{ "columns": ["status", "created_at"], "unique": false }],
      "access_control": {
        "model": "all_systems",
        "read_access": true,
        "update_access": true,
        "insert_access": true,
        "delete_access": true,
        "hard_delete_allowed": true,
        "direct_write_enabled": true
      }
    },
    "createdAt": "2026-09-20T02:11:05.120Z",
    "createdBy": "7d2b9a10-4c3e-4f5a-9b8c-1d2e3f4a5b6c",
    "createdByIdentity": "Jordan Lee",
    "modifiedAt": "2026-09-27T08:40:12.004Z",
    "modifiedBy": "7d2b9a10-4c3e-4f5a-9b8c-1d2e3f4a5b6c",
    "modifiedByIdentity": "Jordan Lee",
    "deletedAt": null
  },
  "meta": {},
  "status": 200,
  "error": false,
  "message": "Config item retrieved successfully"
}
```

**Errors**

| Status | Trigger                                                                                                  | `message` / body                                                                     | Code location                  |
| ------ | -------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------ | ------------------------------ |
| 400    | `slug` empty                                                                                             | `Slug is required`                                                                   | `details[].code` = `too_small` |
| 401    | §3                                                                                                       | per §3                                                                               | —                              |
| 403    | Caller lacks `view` on `domain`                                                                          | `{ "error": "Authorization failed. You don't have access to perform this action." }` | not returned                   |
| 404    | No config item has this slug with `item_type = domain` (including when the slug belongs to another type) | `domain with slug "<slug>" not found`                                                | not returned                   |
| 500    | Lookup failed                                                                                            | `Failed to fetch config item by slug`                                                | not returned                   |

**Example**

```bash
curl "https://pivotly.example.com/api/v3/config-items/domain/by-slug/customer" \
  -H "Authorization: Bearer $TOKEN" -H "X-Tenant-Id: $TENANT_ID"
```

**Notes:** returns the stored (authored) item, not the published configuration — use §9.2–§9.4 or §10.1 for what is published. Also returns soft-deleted (disabled) items (§8.7).

### 8.3 Get a domain by id — GET /v3/config-items/domain/{id}

- **Classification:** consumer.
- **Headers:** `Authorization: Bearer <JWT>` (!), `X-Tenant-Id` (?), `X-App-Slug` (?).
- **Applicable global rules:** §2; §3 (verb `view`), 403 body shape; §4.
- **Check order:** `[REST]` params → `[authz]`.

**Path parameters**

| Param | Type   | Req. | Bounds / format                                                                                                           |
| ----- | ------ | ---- | ------------------------------------------------------------------------------------------------------------------------- |
| `id`  | string | !    | Length ≥ 1 `[REST]`; no uuid check at the request layer — a value that is not a full uuid makes the lookup fail with 500. |

**Success — 200**: `data` is one config item read record — `id` (uuid), `itemType` (`domain`), `isDeleted` (boolean, `false`), `version` (integer), `enabled` (boolean), `parentItemId` (always `null`), `name` (string), `description` (string \| null), `slug` (string), `fullPath` (string), `depth` (integer), `cfgData` (object, stored form; its keys are returned in a fixed order — `schema`, `archive`, `contact`, `multi_tenant`, `xref_enabled`, `system_access`, `access_control`, `archive_policy`, `track_versions`, `domain_table_name`, `slug`, `domain_name_singular`, `default_conflict_resolution` first when present, then the remaining keys in stored order — shortest key first, then alphabetical), `createdAt` / `modifiedAt` (ISO timestamp), `createdBy` / `modifiedBy` (uuid \| null), `createdByIdentity` / `modifiedByIdentity` (string \| null), `deletedAt` (null) — **plus**:

| Field            | Type   | Notes                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| ---------------- | ------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `publish_status` | string | Present only when a publication exists for this slug **at the item's current version**: that publication's `status` (one of `not_set`, `staged`, `preprocessing`, `pending`, `processing`, `complete`, `cancel_requested`). `complete` means the publish run finished — **not** that it succeeded (check the publication's step results, §9.4). Omitted when no publication matches the current version (for example after any save since the last publish). ⚠️ Not determinable from source which publication is reported when several exist for the same version. |

No `message`.

```json
{
  "data": {
    "id": "5b0e8f3a-2c4d-4e6f-8a1b-3c5d7e9f1a2b",
    "itemType": "domain",
    "isDeleted": false,
    "version": 2,
    "enabled": true,
    "parentItemId": null,
    "name": "Customers",
    "description": "Customer master records",
    "slug": "customer",
    "fullPath": "customer",
    "depth": 1,
    "cfgData": {
      "schema": [
        {
          "name": "customer_name",
          "column_name": "customer_name",
          "data_type": "text",
          "nullable": false,
          "triggers_version_change": true
        },
        {
          "name": "email",
          "column_name": "email",
          "data_type": "text",
          "unique": true,
          "triggers_version_change": false
        },
        {
          "name": "status",
          "column_name": "status",
          "data_type": "text",
          "triggers_version_change": false,
          "picklist": { "slug": "pkl-customer-status" }
        }
      ],
      "archive": true,
      "multi_tenant": false,
      "xref_enabled": true,
      "access_control": {
        "model": "all_systems",
        "read_access": true,
        "update_access": true,
        "insert_access": true,
        "delete_access": true,
        "hard_delete_allowed": true,
        "direct_write_enabled": true
      },
      "archive_policy": { "max_record_versions": 10000, "retention_days": 0 },
      "track_versions": true,
      "domain_table_name": "customer",
      "domain_name_singular": "Customer",
      "default_conflict_resolution": { "code": "lww" },
      "area": "CRM",
      "indexes": [{ "columns": ["status", "created_at"], "unique": false }],
      "dedupe_enabled": false
    },
    "createdAt": "2026-09-20T02:11:05.120Z",
    "createdBy": "7d2b9a10-4c3e-4f5a-9b8c-1d2e3f4a5b6c",
    "createdByIdentity": "Jordan Lee",
    "modifiedAt": "2026-09-27T08:40:12.004Z",
    "modifiedBy": "7d2b9a10-4c3e-4f5a-9b8c-1d2e3f4a5b6c",
    "modifiedByIdentity": "Jordan Lee",
    "deletedAt": null,
    "publish_status": "complete"
  },
  "meta": {},
  "status": 200,
  "error": false
}
```

**Errors**

| Status | Trigger                                   | `message` / body                                                                     | Code location                  |
| ------ | ----------------------------------------- | ------------------------------------------------------------------------------------ | ------------------------------ |
| 400    | `id` empty                                | `ID is required`                                                                     | `details[].code` = `too_small` |
| 401    | §3                                        | per §3                                                                               | —                              |
| 403    | Caller lacks `view` on `domain`           | `{ "error": "Authorization failed. You don't have access to perform this action." }` | not returned                   |
| 404    | No item with this id                      | `Config item not found`                                                              | not returned                   |
| 404    | The id belongs to an item of another type | `Config item type mismatch: expected domain, got <type> not found`                   | not returned                   |
| 500    | `id` not a uuid, or lookup failed         | `Failed to fetch config item`                                                        | not returned                   |

**Example**

```bash
curl "https://pivotly.example.com/api/v3/config-items/domain/5b0e8f3a-2c4d-4e6f-8a1b-3c5d7e9f1a2b" \
  -H "Authorization: Bearer $TOKEN" -H "X-Tenant-Id: $TENANT_ID"
```

### 8.4 List domains (paginated) — GET /v3/config-items/domain

- **Classification:** consumer.
- **Headers:** `Authorization: Bearer <JWT>` (!), `X-Tenant-Id` (?), `X-App-Slug` (?).
- **Applicable global rules:** §2; §3 (verb `list`), 403 body shape; §4; §5 pagination/sort/filter conventions (page size 1–100).
- **Check order:** `[REST]` query → `[authz]`.

**Query parameters**

| Param          | Type              | Req. | Bounds / format       | Default                | Notes                                                                                                                                                                                                                                                                                                                   |
| -------------- | ----------------- | ---- | --------------------- | ---------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `fetchType`    | string            | ?    | `list` \| `paginated` | `paginated`            | `list` returns **every** matching row; `page` / `pageSize` are then ignored and returned as `null`.                                                                                                                                                                                                                     |
| `page`         | integer (coerced) | ?    | ≥ 0                   | `0`                    | Zero-based.                                                                                                                                                                                                                                                                                                             |
| `pageSize`     | integer (coerced) | ?    | 1–100                 | `25`                   |                                                                                                                                                                                                                                                                                                                         |
| `slugContains` | string            | ?    | None enforced         | —                      | Case-insensitive substring match on `slug`; `%` and `_` in the value act as wildcards.                                                                                                                                                                                                                                  |
| `search`       | string            | ?    | None enforced         | —                      | Case-insensitive substring match on `name` **or** `slug`; `%` / `_` act as wildcards.                                                                                                                                                                                                                                   |
| `sortModel`    | string (JSON)     | ?    | §5                    | `createdAt` descending | Sortable fields: `id`, `itemType`, `isDeleted`, `version`, `enabled`, `parentItemId`, `name`, `description`, `slug`, `fullPath`, `depth`, `cfgData`, `createdAt`, `createdBy`, `createdByIdentity`, `modifiedAt`, `modifiedBy`, `modifiedByIdentity`, `deletedAt`. Others ignored — `is_published` is **not** sortable. |
| `filterModel`  | string (JSON)     | ?    | §5                    | —                      | Filterable fields: the same 19. Quick-filter fields: `name`, `slug`, `description`, `fullPath`. `is_published` is **not** filterable. Filters are AND-combined with the path type.                                                                                                                                      |

Unknown query parameters are stripped.

**Success — 200**: `data` is an array of reduced records, each with exactly: `id` (uuid), `itemType` (`domain`), `version` (integer), `enabled` (boolean), `name` (string), `description` (string \| null), `slug` (string), `fullPath` (string), `depth` (integer), `cfgData` (object, stored form), `createdAt` / `modifiedAt` (ISO timestamp), `createdBy` / `modifiedBy` (uuid \| null), `parentItemId` (always `null`), and:

| Field          | Type    | Notes                                                                                                                                                                                                                                                                             |
| -------------- | ------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `is_published` | boolean | `true` when a publication exists for this slug **at the item's current version** whose `status` is `complete`. A completed run that **failed** also counts, so `true` does not guarantee a successful publish; any save after publishing makes it `false` until the next publish. |

`isDeleted`, `createdByIdentity`, `modifiedByIdentity` and `deletedAt` are **not** returned here. `pagination` = `{ page, page_size, total_records }`. No `message`.

```json
{
  "data": [
    {
      "id": "5b0e8f3a-2c4d-4e6f-8a1b-3c5d7e9f1a2b",
      "itemType": "domain",
      "version": 2,
      "enabled": true,
      "name": "Customers",
      "description": "Customer master records",
      "slug": "customer",
      "fullPath": "customer",
      "depth": 1,
      "cfgData": {
        "domain_name_singular": "Customer",
        "domain_table_name": "customer",
        "default_conflict_resolution": { "code": "lww" },
        "archive": true,
        "archive_policy": { "max_record_versions": 10000, "retention_days": 0 },
        "xref_enabled": true,
        "track_versions": true,
        "multi_tenant": false,
        "dedupe_enabled": false,
        "area": "CRM",
        "schema": [
          {
            "name": "customer_name",
            "column_name": "customer_name",
            "data_type": "text",
            "nullable": false,
            "triggers_version_change": true
          },
          {
            "name": "email",
            "column_name": "email",
            "data_type": "text",
            "unique": true,
            "triggers_version_change": false
          },
          {
            "name": "status",
            "column_name": "status",
            "data_type": "text",
            "triggers_version_change": false,
            "picklist": { "slug": "pkl-customer-status" }
          }
        ],
        "indexes": [{ "columns": ["status", "created_at"], "unique": false }],
        "access_control": {
          "model": "all_systems",
          "read_access": true,
          "update_access": true,
          "insert_access": true,
          "delete_access": true,
          "hard_delete_allowed": true,
          "direct_write_enabled": true
        }
      },
      "createdAt": "2026-09-20T02:11:05.120Z",
      "createdBy": "7d2b9a10-4c3e-4f5a-9b8c-1d2e3f4a5b6c",
      "modifiedAt": "2026-09-27T08:40:12.004Z",
      "modifiedBy": "7d2b9a10-4c3e-4f5a-9b8c-1d2e3f4a5b6c",
      "parentItemId": null,
      "is_published": true
    }
  ],
  "meta": {},
  "status": 200,
  "error": false,
  "pagination": { "page": 0, "page_size": 25, "total_records": 1 }
}
```

**Errors**

| Status | Trigger                                                              | `message` / body                                                                     | Code location            |
| ------ | -------------------------------------------------------------------- | ------------------------------------------------------------------------------------ | ------------------------ |
| 400    | Invalid `fetchType`, non-numeric or out-of-range `page` / `pageSize` | issue message or `Schema validation error`                                           | `details[].code` (§11.2) |
| 401    | §3                                                                   | per §3                                                                               | —                        |
| 403    | Caller lacks `list` on `domain`                                      | `{ "error": "Authorization failed. You don't have access to perform this action." }` | not returned             |
| 500    | Query failed (e.g. a filter value the field's type cannot accept)    | `Failed to fetch config items`                                                       | not returned             |

Malformed `sortModel` / `filterModel` JSON is **not** an error — it is ignored (§5).

**Example**

```bash
curl -G "https://pivotly.example.com/api/v3/config-items/domain" \
  -H "Authorization: Bearer $TOKEN" -H "X-Tenant-Id: $TENANT_ID" \
  --data-urlencode "page=0" --data-urlencode "pageSize=25" \
  --data-urlencode "search=cust" \
  --data-urlencode 'sortModel=[{"field":"modifiedAt","sort":"desc"}]'
```

### 8.5 Create a domain — POST /v3/config-items/domain

- **Classification:** consumer.
- **Headers:** `Authorization: Bearer <JWT>` (!), `Content-Type: application/json` (!), `X-Tenant-Id` (?), `X-App-Slug` (?).
- **Applicable global rules:** §2; §3 (verb `create`), 403 body shape; §4; §6 (every item-envelope and `cfg_data` rule, the §6.3 injections); §7 slug pattern, reserved names and prefixes, enumerations; §11 (save failures surface as 500).
- **Check order:** `[REST]` body → `[authz]` → `[service]` type match → `[service]` no id → `[save-fn]` slug availability → `[save-fn]` defaults and injections → `[DB-validator]` → `[DB-constraint]` → stored.

**Request body**

```json
{
  "parameters": {},
  "data": {
    "slug": "string",
    "item_type": "domain",
    "name": "string",
    "description": "string | null",
    "enabled": true,
    "full_path": "string",
    "parent_item_id": "uuid",
    "cfg_data": {}
  }
}
```

| Field                  | Type            | Req.           | Default                        | Combined rule (exact)                                                                                                                                                                                                                                                                                                                                                                        | Layer / stage                                                                                                                                           | Description                                  |
| ---------------------- | --------------- | -------------- | ------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------- |
| `parameters`           | object          | ?              | —                              | Object if present. Its only recognized key is `id`, which is **forbidden** here; other keys are stripped.                                                                                                                                                                                                                                                                                    | `[REST]`, `[service]`                                                                                                                                   | Operation parameters.                        |
| `parameters.id`        | string (uuid)   | must be absent | —                              | Present → 400 `ID should not be provided for creation. Use PATCH for updates.`                                                                                                                                                                                                                                                                                                               | `[service]`                                                                                                                                             | —                                            |
| `data`                 | object          | !              | —                              | Object.                                                                                                                                                                                                                                                                                                                                                                                      | `[REST]`                                                                                                                                                | The item.                                    |
| `data.slug`            | string          | !              | —                              | `^[a-z](?:[a-z0-9_]*[a-z0-9])?$`, 1–50 chars; not starting `sys_`, `dvw_`, `dtm_`, `dtv_`; unused by **any** config item of any type; equal to `cfg_data.domain_table_name`. `[REST]` accepts 1–255 chars, so 51–255 chars passes the request layer and is rejected at save (500, `VALUE`); > 255 → 400. A slug starting with a digit passes the validator but is rejected by storage (500). | `[REST]` 1–255; `[save-fn]` uniqueness; `[DB-validator]` `VALUE`, `RESERVED_PREFIX_SLUG`, `RESERVED_PREFIX_USDF`, `SLUG_TABLE_MATCH`; `[DB-constraint]` | Globally unique identifier and storage name. |
| `data.item_type`       | string          | !              | —                              | Exactly `domain` (must equal the path segment); `[REST]` 1–100 chars.                                                                                                                                                                                                                                                                                                                        | `[REST]`, `[service]`                                                                                                                                   |                                              |
| `data.cfg_data`        | object          | !              | —                              | JSON object (arrays rejected at `[REST]`). Must satisfy §6.2–§6.8: at minimum `domain_name_singular`, `domain_table_name` (= slug), `default_conflict_resolution.code` and a non-empty `schema` whose every entry has `name`, `column_name`, `data_type` and a boolean `triggers_version_change`. `access_control` is overwritten and `system_access` normalized (§6.3).                     | `[REST]`, `[save-fn]`, `[DB-validator]`                                                                                                                 | The domain definition.                       |
| `data.name`            | string          | ?              | the slug (when absent or `""`) | ≤ 255 chars; `null` rejected (400). Not trimmed on create.                                                                                                                                                                                                                                                                                                                                   | `[REST]`; `[save-fn]` default; `[DB-validator]` `REQUIRED` (never fires on create because of the default)                                               | Display name.                                |
| `data.description`     | string \| null  | ?              | `null`                         | ≤ 1000 chars. No format. Stored as sent.                                                                                                                                                                                                                                                                                                                                                     | `[REST]`                                                                                                                                                |                                              |
| `data.enabled`         | boolean \| null | ?              | `true` (also when `null`)      | Boolean.                                                                                                                                                                                                                                                                                                                                                                                     | `[REST]`; `[save-fn]` default                                                                                                                           |                                              |
| `data.full_path`       | string          | ?              | the slug (when absent or `""`) | ≤ 500 chars; `null` rejected (400). No format enforced. Not trimmed on create.                                                                                                                                                                                                                                                                                                               | `[REST]`; `[save-fn]` default                                                                                                                           | Hierarchy path; drives `depth`.              |
| `data.parent_item_id`  | string (uuid)   | ?              | `null`                         | `^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$` (case-insensitive); `null` rejected (400). Existence not checked.                                                                                                                                                                                                                                                           | `[REST]`                                                                                                                                                |                                              |
| `data.version`         | integer         | ? — ignored    | 1                              | Positive integer if sent (else 400); ignored.                                                                                                                                                                                                                                                                                                                                                | `[REST]`                                                                                                                                                | Server-set.                                  |
| `data.id`              | string (uuid)   | must be absent | —                              | Present → 400 (as `parameters.id`).                                                                                                                                                                                                                                                                                                                                                          | `[service]`                                                                                                                                             | Server-generated.                            |
| any other `data.*` key | —               | —              | —                              | Stripped (e.g. `is_deleted`).                                                                                                                                                                                                                                                                                                                                                                | `[REST]`                                                                                                                                                |                                              |

**Effective request contract**

- **Required to send:** `data.slug`, `data.item_type` (= `domain`), `data.cfg_data` with, at minimum, `domain_name_singular`, `domain_table_name` (= `data.slug`), `default_conflict_resolution: { code }` and a non-empty `schema[]` (each entry: `name`, `column_name`, `data_type`, `triggers_version_change`).
- **Optional / defaulted (applied before the validator runs):** `data.name` → slug; `data.full_path` → slug; `data.enabled` → `true`; `data.description` → `null`; `data.parent_item_id` → `null`; `parameters` → `{}`. `name` is required by the validator but is effectively optional because of this default. Recommended although optional: `archive` (+ `archive_policy`), `xref_enabled`, `track_versions` (see the §6.2 warning).
- **Ignored / overwritten / server-set:** `id` (generated; sending it is an error), `version` (set to 1), `is_deleted` (stripped; stored `false`), `cfg_data.access_control` (replaced, §6.3), unknown keys of `system_access` entries (removed, §6.3), `created_*` / `modified_*`, `depth`.

**Signature**

`create(data{ slug!, item_type!("domain"), cfg_data!{ domain_name_singular!, domain_table_name!(=slug), default_conflict_resolution!{ code! }, schema![{ name!, column_name!, data_type!, triggers_version_change! ; … }] ; archive?, archive_policy?, xref_enabled?, track_versions?, multi_tenant?, dedupe_enabled?, area?, read_view_slug?, system_access?, indexes?, contact? } ; name?, description?, enabled?, full_path?, parent_item_id? } ; parameters?{}) — id forbidden/server-set, version ignored (=1), access_control overwritten`

**Validation rules** (defined in §6; listed here with their stage)

1. `[REST]` body shape and bounds (400).
2. `[service]` `data.item_type` = `domain` (400); no `data.id` / `parameters.id` (400).
3. `[save-fn]` slug availability: if any config item already has `data.slug` → 500 (two messages, see errors).
4. `[save-fn]` §6.3 injections.
5. `[DB-validator]` — all failures collected into one 500 `Validation failed` response with `details[]`: every code in §11.3 whose rule applies (envelope `REQUIRED`/`VALUE`, `RESERVED_PREFIX_SLUG`, `RESERVED_PREFIX_USDF`, `TYPE`, `SLUG_TABLE_MATCH`, `XREF_REQUIRES_TRACK`, `ARCHIVE_POLICY_REQUIRED`, `INVALID_READ_VIEW_SLUG`, the `schema[]` item, collection, `system_access[]` and `indexes[]` codes, and — when the slug is already in the published-domain registry (e.g. a re-created domain) — the §6.9 immutability codes). When `cfg_data` is not an object only the envelope checks and `TYPE` run.
6. `[DB-constraint]` slug pattern `^[a-z][a-z0-9_-]{0,127}$` (500).

**Success — 201**

```json
{
  "data": {
    "id": "5b0e8f3a-2c4d-4e6f-8a1b-3c5d7e9f1a2b",
    "item_type": "domain",
    "is_deleted": false,
    "deleted_at": null,
    "version": 1,
    "enabled": true,
    "parent_item_id": null,
    "name": "Customers",
    "description": "Customer master records",
    "slug": "customer",
    "full_path": "customer",
    "depth": 1,
    "cfg_data": {
      "domain_name_singular": "Customer",
      "domain_table_name": "customer",
      "default_conflict_resolution": { "code": "lww" },
      "archive": true,
      "archive_policy": { "max_record_versions": 10000, "retention_days": 0 },
      "xref_enabled": true,
      "track_versions": true,
      "multi_tenant": false,
      "dedupe_enabled": false,
      "area": "CRM",
      "schema": [
        {
          "name": "customer_name",
          "column_name": "customer_name",
          "data_type": "text",
          "nullable": false,
          "triggers_version_change": true
        },
        {
          "name": "email",
          "column_name": "email",
          "data_type": "text",
          "unique": true,
          "triggers_version_change": false
        },
        {
          "name": "status",
          "column_name": "status",
          "data_type": "text",
          "triggers_version_change": false,
          "picklist": { "slug": "pkl-customer-status" }
        }
      ],
      "indexes": [{ "columns": ["status", "created_at"], "unique": false }],
      "access_control": {
        "model": "all_systems",
        "read_access": true,
        "update_access": true,
        "insert_access": true,
        "delete_access": true,
        "hard_delete_allowed": true,
        "direct_write_enabled": true
      }
    },
    "created_at": "2026-09-28T01:15:22.310412+00:00",
    "created_by": "7d2b9a10-4c3e-4f5a-9b8c-1d2e3f4a5b6c",
    "modified_at": "2026-09-28T01:15:22.310412+00:00",
    "modified_by": "7d2b9a10-4c3e-4f5a-9b8c-1d2e3f4a5b6c"
  },
  "meta": {
    "action": "insert",
    "id": "5b0e8f3a-2c4d-4e6f-8a1b-3c5d7e9f1a2b",
    "slug": "customer",
    "version": 1,
    "item_type": "domain"
  },
  "status": 201,
  "error": false,
  "message": "success"
}
```

`data` is the save response row: `id` (uuid), `item_type`, `is_deleted` (boolean), `deleted_at` (null), `version` (integer), `enabled` (boolean), `parent_item_id` (uuid \| null), `name`, `description` (string \| null), `slug`, `full_path`, `depth` (integer), `cfg_data` (object, stored form), `created_at`, `created_by` (uuid \| null), `modified_at`, `modified_by` (uuid \| null). `meta` = `{ action: "insert", id, slug, version, item_type }`.

**Errors**

| Status | Trigger                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | `message`                                                                                                                 | Code location                                                       |
| ------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------- |
| 400    | Request shape: `data` missing/not an object; `slug` empty (`Slug is required`), missing or not a string (`invalid_type`, the validator's default "Invalid input: expected string…" text) or > 255 (`Slug must be less than 255 characters`); `item_type` empty (`Item type is required`), missing or not a string (`invalid_type`) or > 100 (`Item type must be less than 100 characters`); `cfg_data` missing or not an object; `name` > 255 (`Name must be less than 255 characters`) or `null`; `description` > 1000 (`Description must be less than 1000 characters`); `full_path` > 500 (`Full path must be less than 500 characters`) or `null`; `id` / `parent_item_id` not a uuid (`Invalid UUID format`); `version` not a positive integer; `enabled` not boolean/null | single issue message, or `Schema validation error` for several                                                            | `details[].code` (§11.2)                                            |
| 400    | Malformed JSON body                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | the JSON parser's text                                                                                                    | no `details`                                                        |
| 400    | `data.item_type` ≠ `domain`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | `Item type mismatch: URL specifies "domain" but body specifies "<item_type>".`                                            | not returned                                                        |
| 400    | `data.id` or `parameters.id` present                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | `ID should not be provided for creation. Use PATCH for updates.`                                                          | not returned                                                        |
| 401    | §3                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | per §3                                                                                                                    | —                                                                   |
| 403    | Caller lacks `create` on `domain`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | `{ "error": "Authorization failed. You don't have access to perform this action." }`                                      | not returned                                                        |
| 500    | Slug already used by another **domain**                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | message states the slug already exists and that updating requires the existing item's id (which it quotes)                | not returned; `meta` = `{ action, id, slug, item_type }`            |
| 500    | Slug already used by an item of **another type**                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | `Slug "<slug>" already exists with item_type "<type>"; cannot save as "domain".`                                          | not returned; `meta` as above                                       |
| 500    | Domain validation failed                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | `Validation failed` (or, if the validator itself errors, the underlying error text with a single `INVALID` entry — §11.3) | `details[].code` — one entry per violation (§11.3); `meta` as above |
| 500    | Slug passes the validator but fails the storage pattern (starts with a digit)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | storage check-constraint violation message                                                                                | not returned; `meta` as above                                       |

**Example**

```bash
curl -X POST "https://pivotly.example.com/api/v3/config-items/domain" \
  -H "Authorization: Bearer $TOKEN" -H "X-Tenant-Id: $TENANT_ID" \
  -H "Content-Type: application/json" \
  -d '{
    "data": {
      "slug": "customer",
      "item_type": "domain",
      "name": "Customers",
      "description": "Customer master records",
      "cfg_data": {
        "domain_name_singular": "Customer",
        "domain_table_name": "customer",
        "default_conflict_resolution": {
          "code": "lww"
        },
        "archive": true,
        "archive_policy": {
          "max_record_versions": 10000,
          "retention_days": 0
        },
        "xref_enabled": true,
        "track_versions": true,
        "multi_tenant": false,
        "dedupe_enabled": false,
        "area": "CRM",
        "schema": [
          {
            "name": "customer_name",
            "column_name": "customer_name",
            "data_type": "text",
            "nullable": false,
            "triggers_version_change": true
          },
          {
            "name": "email",
            "column_name": "email",
            "data_type": "text",
            "unique": true,
            "triggers_version_change": false
          },
          {
            "name": "status",
            "column_name": "status",
            "data_type": "text",
            "triggers_version_change": false,
            "picklist": {
              "slug": "pkl-customer-status"
            }
          }
        ],
        "indexes": [
          {
            "columns": [
              "status",
              "created_at"
            ],
            "unique": false
          }
        ]
      }
    }
  }'
```

**Notes:** saving never creates or alters storage — call §9.1 to publish. Not idempotent: repeating the call fails on the second attempt (slug taken).

### 8.6 Update a domain — PATCH /v3/config-items/domain/{id}

- **Classification:** consumer.
- **Headers:** `Authorization: Bearer <JWT>` (!), `Content-Type: application/json` (!), `X-Tenant-Id` (?), `X-App-Slug` (?).
- **Applicable global rules:** §2; §3 (verb `change`), 403 body shape; §4; §6 (every rule, including the §6.3 injections and §6.9 immutability); §7; §11.
- **Check order:** `[REST]` body + path → `[authz]` → `[service]` type match → `[service]` id match → `[save-fn]` load by id, patch-merge, type and slug rules, injections → `[DB-validator]` → `[DB-constraint]` → stored.

**Path parameters**

| Param | Type   | Req. | Bounds / format                                                                                                              |
| ----- | ------ | ---- | ---------------------------------------------------------------------------------------------------------------------------- |
| `id`  | string | !    | Length ≥ 1 `[REST]`. Must be a full uuid: any other value makes the request fail with 500 (generic `Internal server error`). |

**Request body** — the update route validates the **same body shape as create**: `slug`, `item_type` and `cfg_data` are required on every update.

```json
{
  "parameters": { "id": "uuid (optional, must equal the path id)" },
  "data": {
    "id": "uuid (optional, must equal the path id)",
    "slug": "string (the current slug, or a new one while never published)",
    "item_type": "domain",
    "cfg_data": {},
    "name": "string",
    "description": "string | null",
    "enabled": true,
    "full_path": "string",
    "parent_item_id": "uuid"
  }
}
```

| Field                  | Type            | Req.        | Omitted →           | Combined rule (exact)                                                                                                                                                                                                                                                                                                           | Layer / stage                                              |
| ---------------------- | --------------- | ----------- | ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------- |
| `parameters`           | object          | ?           | —                   | Only `id` recognized; other keys stripped.                                                                                                                                                                                                                                                                                      | `[REST]`                                                   |
| `parameters.id`        | string (uuid)   | ?           | —                   | Uuid format; if present must equal path `id` (else 400 `ID mismatch between URL and body.`).                                                                                                                                                                                                                                    | `[REST]`, `[service]`                                      |
| `data`                 | object          | !           | —                   | Object.                                                                                                                                                                                                                                                                                                                         | `[REST]`                                                   |
| `data.id`              | string (uuid)   | ?           | —                   | As `parameters.id`.                                                                                                                                                                                                                                                                                                             | `[REST]`, `[service]`                                      |
| `data.slug`            | string          | !           | (cannot be omitted) | 1–255 chars `[REST]`; trimmed. Equal to the stored slug, **or** a new slug only while the stored slug is not in the published-domain registry (else 500 — slug rename blocked, `IMMUTABLE_SLUG` in the message). A new slug must satisfy every §6.1 rule, be unused (else 500), and `cfg_data.domain_table_name` must equal it. | `[REST]`, `[save-fn]`, `[DB-validator]`, `[DB-constraint]` |
| `data.item_type`       | string          | !           | (cannot be omitted) | Exactly `domain` (= path; 400 otherwise); must equal the stored type (500 `Item type conflict for slug "<slug>" (existing="<type>", provided="domain")` when the id belongs to another type).                                                                                                                                   | `[REST]`, `[service]`, `[save-fn]`                         |
| `data.cfg_data`        | object          | !           | (cannot be omitted) | JSON object; **replaces the stored `cfg_data` wholesale** (no key-level merge), then the §6.3 injections are applied. Must be the complete, valid definition; all §6.2–§6.9 rules apply, including immutability against the published configuration.                                                                            | `[REST]`, `[save-fn]`, `[DB-validator]`                    |
| `data.name`            | string          | ?           | keeps stored value  | ≤ 255 chars; `null` → 400. Trimmed; an empty or whitespace-only value is rejected (500, `REQUIRED` on `$.name`).                                                                                                                                                                                                                | `[REST]`, `[save-fn]`, `[DB-validator]`                    |
| `data.description`     | string \| null  | ?           | keeps stored value  | ≤ 1000 chars. Trimmed; `null`, `""` or whitespace-only clears it to `null`.                                                                                                                                                                                                                                                     | `[REST]`, `[save-fn]`                                      |
| `data.enabled`         | boolean \| null | ?           | keeps stored value  | Boolean. `null` → 500 (storage not-null violation).                                                                                                                                                                                                                                                                             | `[REST]`, `[DB-constraint]`                                |
| `data.full_path`       | string          | ?           | keeps stored value  | ≤ 500 chars; `null` → 400. Trimmed; `""` or whitespace-only → 500 (storage not-null violation). No format.                                                                                                                                                                                                                      | `[REST]`, `[DB-constraint]`                                |
| `data.parent_item_id`  | string (uuid)   | ?           | keeps stored value  | Uuid pattern as in §8.5; `null` → 400, so it cannot be cleared through this endpoint. Existence not checked.                                                                                                                                                                                                                    | `[REST]`                                                   |
| `data.version`         | integer         | ? — ignored | —                   | Positive integer if sent (else 400); ignored.                                                                                                                                                                                                                                                                                   | `[REST]`                                                   |
| any other `data.*` key | —               | —           | —                   | Stripped.                                                                                                                                                                                                                                                                                                                       | `[REST]`                                                   |

**Effective request contract (update)**

- **Required to send:** path `id` (uuid); `data.slug`; `data.item_type` = `domain`; `data.cfg_data` — the **complete** definition. The request layer requires these three body fields on every update, even though the platform would otherwise keep stored values: **update is a full resubmission of `slug`, `item_type` and `cfg_data`, targeted by the path id.** To change only `name`, re-send the stored `slug`, `item_type` and `cfg_data` (the stored `cfg_data` may be re-sent as read, including the injected `access_control`) together with the new `name`.
- **Optional (patch-merged):** `name`, `description`, `enabled`, `full_path`, `parent_item_id` — omitted fields keep their stored values; present fields replace them (text values trimmed; see the table for `null` / empty behaviour).
- **Merge behaviour of JSON fields:** `cfg_data` is replaced wholesale — no "omit to keep stored", no key-level merge — and then re-injected (§6.3).
- **Concurrency:** none. A sent `version` is ignored; every successful update increments the stored version by 1 (last write wins; no optimistic-lock check).
- **Immutability:** `item_type` never changes; `slug` only while never published; once published, §6.9 applies (one-way capability flags, attribute data types).
- **Unknown id:** if no config item has the path `id`, the platform **creates** a new domain with that id instead of failing (`meta.action = "insert"`, `version` 1, status 200). Create rules apply (the slug must be unused; `name` / `full_path` default to the slug; `enabled` defaults to `true`). Check existence with §8.3 first if an update-only behaviour is needed.
- **Side effects:** the item's soft-delete state is reset to not-deleted on every update (it is never set by this API — §8.7). The version increment means the domain no longer counts as published at its current version (`is_published: false` in §8.4, no `publish_status` in §8.3) until §9.1 is called again. Storage is unchanged until then.
- **Ignored / server-set:** `version`, `cfg_data.access_control` (overwritten), `modified_at`, `modified_by`, `depth`, unknown keys.

**Signature**

`update(path id!(uuid) ; data{ slug!(current, or new while unpublished), item_type!("domain", immutable), cfg_data!(complete, whole-replace, re-injected) ; name?, description?, enabled?(not null), full_path?(not empty), parent_item_id?, id?(=path) } ; parameters?{ id?(=path) }) — version ignored/auto-increment, no optimistic lock, unknown id creates`

**Validation rules**

1. `[REST]` body shape and bounds, identical to create (400).
2. `[service]` `data.item_type` = `domain` (400); `parameters.id` / `data.id`, when present, equal the path id (400).
3. `[save-fn]` load by id; `item_type` unchanged (500); slug unchanged or renamable (500 when the current slug is published).
4. `[save-fn]` §6.3 injections.
5. `[DB-validator]` every rule in §8.5 rule 5, on the merged record (500 `Validation failed` with `details[]`), including the §6.9 immutability rules for a published slug.
6. `[DB-constraint]` `enabled` not null; `full_path` not empty; slug pattern and uniqueness on rename (500).

**Success — 200**

```json
{
  "data": {
    "id": "5b0e8f3a-2c4d-4e6f-8a1b-3c5d7e9f1a2b",
    "item_type": "domain",
    "is_deleted": false,
    "deleted_at": null,
    "version": 2,
    "enabled": true,
    "parent_item_id": null,
    "name": "Customers",
    "description": "Customer master records",
    "slug": "customer",
    "full_path": "customer",
    "depth": 1,
    "cfg_data": {
      "domain_name_singular": "Customer",
      "domain_table_name": "customer",
      "default_conflict_resolution": { "code": "lww" },
      "archive": true,
      "archive_policy": { "max_record_versions": 10000, "retention_days": 0 },
      "xref_enabled": true,
      "track_versions": true,
      "multi_tenant": false,
      "dedupe_enabled": false,
      "area": "CRM",
      "schema": [
        {
          "name": "customer_name",
          "column_name": "customer_name",
          "data_type": "text",
          "nullable": false,
          "triggers_version_change": true
        },
        {
          "name": "email",
          "column_name": "email",
          "data_type": "text",
          "unique": true,
          "triggers_version_change": false
        },
        {
          "name": "status",
          "column_name": "status",
          "data_type": "text",
          "triggers_version_change": false,
          "picklist": { "slug": "pkl-customer-status" }
        }
      ],
      "indexes": [{ "columns": ["status", "created_at"], "unique": false }],
      "access_control": {
        "model": "all_systems",
        "read_access": true,
        "update_access": true,
        "insert_access": true,
        "delete_access": true,
        "hard_delete_allowed": true,
        "direct_write_enabled": true
      }
    },
    "created_at": "2026-09-28T01:15:22.310412+00:00",
    "created_by": "7d2b9a10-4c3e-4f5a-9b8c-1d2e3f4a5b6c",
    "modified_at": "2026-09-28T02:03:41.902118+00:00",
    "modified_by": "7d2b9a10-4c3e-4f5a-9b8c-1d2e3f4a5b6c"
  },
  "meta": {
    "action": "update",
    "id": "5b0e8f3a-2c4d-4e6f-8a1b-3c5d7e9f1a2b",
    "slug": "customer",
    "version": 2,
    "item_type": "domain"
  },
  "status": 200,
  "error": false,
  "message": "success"
}
```

`data` is the save response row: `id` (uuid), `item_type`, `is_deleted` (boolean), `deleted_at` (null), `version` (integer), `enabled` (boolean), `parent_item_id` (uuid \| null), `name`, `description` (string \| null), `slug`, `full_path`, `depth` (integer), `cfg_data` (object, stored form), `created_at`, `created_by` (uuid \| null), `modified_at`, `modified_by` (uuid \| null). `meta` = `{ action: "update" | "insert", id, slug, version, item_type }`.

**Errors**

| Status | Trigger                                                                                                                                                    | `message`                                                                                                                 | Code location                                                         |
| ------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------- |
| 400    | Request shape — every create-body failure listed in §8.5 (missing `slug` / `item_type` / `cfg_data` included), or `id` path param empty (`ID is required`) | issue message or `Schema validation error`                                                                                | `details[].code` (§11.2)                                              |
| 400    | `data.item_type` ≠ `domain`                                                                                                                                | `Item type mismatch: URL specifies "domain" but body specifies "<item_type>".`                                            | not returned                                                          |
| 400    | `parameters.id` or `data.id` ≠ path id                                                                                                                     | `ID mismatch between URL and body.`                                                                                       | not returned                                                          |
| 401    | §3                                                                                                                                                         | per §3                                                                                                                    | —                                                                     |
| 403    | Caller lacks `change` on `domain`                                                                                                                          | `{ "error": "Authorization failed. You don't have access to perform this action." }`                                      | not returned                                                          |
| 500    | Path `id` is not a uuid                                                                                                                                    | `Internal server error` (production; development deployments return the underlying message)                               | not returned                                                          |
| 500    | Slug changed while the current slug is published                                                                                                           | message states the rename is blocked because the domain is published and ends `Code: IMMUTABLE_SLUG`                      | inside `message` (suffix); `meta` = `{ action, id, slug, item_type }` |
| 500    | Id belongs to another item type                                                                                                                            | `Item type conflict for slug "<slug>" (existing="<type>", provided="domain")`                                             | not returned; `meta` as above                                         |
| 500    | Renamed to a slug that is already used                                                                                                                     | storage unique-constraint violation message                                                                               | not returned; `meta` as above                                         |
| 500    | Domain validation failed                                                                                                                                   | `Validation failed` (or, if the validator itself errors, the underlying error text with a single `INVALID` entry — §11.3) | `details[].code` (§11.3); `meta` as above                             |
| 500    | `enabled: null`, or `full_path` empty                                                                                                                      | storage not-null violation message naming the column                                                                      | not returned; `meta` as above                                         |
| 500    | Unknown id and the slug is already used (the create path, §8.5)                                                                                            | the two slug-taken messages of §8.5                                                                                       | not returned; `meta` as above                                         |

**Example**

```bash
curl -X PATCH "https://pivotly.example.com/api/v3/config-items/domain/5b0e8f3a-2c4d-4e6f-8a1b-3c5d7e9f1a2b" \
  -H "Authorization: Bearer $TOKEN" -H "X-Tenant-Id: $TENANT_ID" \
  -H "Content-Type: application/json" \
  -d '{
    "data": {
      "slug": "customer",
      "item_type": "domain",
      "name": "Customers",
      "cfg_data": {
        "domain_name_singular": "Customer",
        "domain_table_name": "customer",
        "default_conflict_resolution": {
          "code": "lww"
        },
        "archive": true,
        "archive_policy": {
          "max_record_versions": 10000,
          "retention_days": 0
        },
        "xref_enabled": true,
        "track_versions": true,
        "multi_tenant": false,
        "dedupe_enabled": false,
        "area": "CRM",
        "schema": [
          {
            "name": "customer_name",
            "column_name": "customer_name",
            "data_type": "text",
            "nullable": false,
            "triggers_version_change": true
          },
          {
            "name": "email",
            "column_name": "email",
            "data_type": "text",
            "unique": true,
            "triggers_version_change": false
          },
          {
            "name": "status",
            "column_name": "status",
            "data_type": "text",
            "triggers_version_change": false,
            "picklist": {
              "slug": "pkl-customer-status"
            }
          }
        ],
        "indexes": [
          {
            "columns": [
              "status",
              "created_at"
            ],
            "unique": false
          }
        ]
      }
    }
  }'
```

### 8.7 Soft-delete a domain — DELETE /v3/config-items/domain/{id}

- **Classification:** consumer.
- **Headers:** `Authorization: Bearer <JWT>` (!), `Content-Type: application/json` (! only when a body is sent), `X-Tenant-Id` (?), `X-App-Slug` (?).
- **Applicable global rules:** §2; §3 (verb `delete`), 403 body shape; §4; §6 (the injections and the validator re-run on the resulting record); §11.
- **Check order:** `[REST]` body + path → `[authz]` → `[service]` load existing (404) → `[save-fn]` update → `[DB-validator]` → `[DB-constraint]` → stored.

> **What this endpoint actually does.** It saves the item again with `enabled = false`, incrementing its version. The item is **not** marked deleted: afterwards it still has `isDeleted: false`, `deletedAt: null`, still appears in §8.1–§8.4, can be updated (which does not re-enable it unless `enabled: true` is sent) and **can still be published** (§9.1). It does **not** remove or change the domain's published storage, its records, its publications or its entry in §10.1.

**Path parameters**

| Param | Type   | Req. | Bounds / format                                                           |
| ----- | ------ | ---- | ------------------------------------------------------------------------- |
| `id`  | string | !    | Length ≥ 1 `[REST]`. Not a full uuid → 500 `Failed to fetch config item`. |

**Request body** — optional. Every field is optional; each is validated with the same per-field format rules as create.

| Field                 | Type            | Req. | Behaviour when sent                                                                      | Behaviour when omitted                                                          | Layer                                   |
| --------------------- | --------------- | ---- | ---------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------- | --------------------------------------- |
| `parameters.id`       | string (uuid)   | ?    | Format-checked, then **ignored** (the path id is used)                                   | —                                                                               | `[REST]`                                |
| `data.id`             | string (uuid)   | ?    | Format-checked, then ignored                                                             | —                                                                               | `[REST]`                                |
| `data.slug`           | string          | ?    | 1–255 chars; treated like an update's slug (a different slug is a rename — §8.6 rules)   | stored slug is used                                                             | `[REST]`, `[save-fn]`                   |
| `data.item_type`      | string          | ?    | 1–100 chars; then **ignored** (the path type is used)                                    | —                                                                               | `[REST]`                                |
| `data.cfg_data`       | object          | ?    | Replaces the stored `cfg_data` wholesale, is re-injected (§6.3) and must pass validation | stored `cfg_data` is re-submitted unchanged (then re-injected and re-validated) | `[REST]`, `[save-fn]`, `[DB-validator]` |
| `data.name`           | string          | ?    | As update (§8.6)                                                                         | stored value kept                                                               | `[REST]`, `[save-fn]`                   |
| `data.description`    | string \| null  | ?    | As update                                                                                | stored value kept                                                               | `[REST]`, `[save-fn]`                   |
| `data.full_path`      | string          | ?    | As update                                                                                | stored value kept                                                               | `[REST]`, `[save-fn]`                   |
| `data.parent_item_id` | string (uuid)   | ?    | As update                                                                                | stored value kept                                                               | `[REST]`                                |
| `data.enabled`        | boolean \| null | ?    | Format-checked, then **overridden** to `false`                                           | —                                                                               | `[REST]`                                |
| `data.version`        | integer         | ?    | Format-checked, then ignored                                                             | —                                                                               | `[REST]`                                |

**Effective request contract (soft-delete)**

- **Required to send:** path `id` (uuid). No body is needed.
- **Optional:** the body fields above; sending `data.cfg_data` overwrites the stored definition.
- **Ignored / overridden:** `parameters.id`, `data.id`, `data.item_type`, `data.enabled` (forced `false`), `data.version`; the deleted flag the handler requests is ignored by the platform.
- **Concurrency:** none; version auto-increments.
- **Validation re-runs:** the stored (or sent) `cfg_data` is validated again, so a soft-delete can fail with `Validation failed` if the stored definition violates a current rule.

**Signature**

`softDelete(path id!(uuid) ; body?{ data?{ slug?(=current), cfg_data?(whole-replace), name?, description?, full_path?, parent_item_id? } }) — sets enabled=false; version auto-increment; not marked deleted; storage and publications untouched; id/item_type/enabled/version in body ignored`

**Success — 200**

```json
{
  "data": {
    "id": "5b0e8f3a-2c4d-4e6f-8a1b-3c5d7e9f1a2b",
    "item_type": "domain",
    "is_deleted": false,
    "deleted_at": null,
    "version": 3,
    "enabled": false,
    "parent_item_id": null,
    "name": "Customers",
    "description": "Customer master records",
    "slug": "customer",
    "full_path": "customer",
    "depth": 1,
    "cfg_data": {
      "domain_name_singular": "Customer",
      "domain_table_name": "customer",
      "default_conflict_resolution": { "code": "lww" },
      "archive": true,
      "archive_policy": { "max_record_versions": 10000, "retention_days": 0 },
      "xref_enabled": true,
      "track_versions": true,
      "multi_tenant": false,
      "dedupe_enabled": false,
      "area": "CRM",
      "schema": [
        {
          "name": "customer_name",
          "column_name": "customer_name",
          "data_type": "text",
          "nullable": false,
          "triggers_version_change": true
        },
        {
          "name": "email",
          "column_name": "email",
          "data_type": "text",
          "unique": true,
          "triggers_version_change": false
        },
        {
          "name": "status",
          "column_name": "status",
          "data_type": "text",
          "triggers_version_change": false,
          "picklist": { "slug": "pkl-customer-status" }
        }
      ],
      "indexes": [{ "columns": ["status", "created_at"], "unique": false }],
      "access_control": {
        "model": "all_systems",
        "read_access": true,
        "update_access": true,
        "insert_access": true,
        "delete_access": true,
        "hard_delete_allowed": true,
        "direct_write_enabled": true
      }
    },
    "created_at": "2026-09-28T01:15:22.310412+00:00",
    "created_by": "7d2b9a10-4c3e-4f5a-9b8c-1d2e3f4a5b6c",
    "modified_at": "2026-09-28T03:30:05.441907+00:00",
    "modified_by": "7d2b9a10-4c3e-4f5a-9b8c-1d2e3f4a5b6c"
  },
  "meta": {
    "action": "update",
    "id": "5b0e8f3a-2c4d-4e6f-8a1b-3c5d7e9f1a2b",
    "slug": "customer",
    "version": 3,
    "item_type": "domain"
  },
  "status": 200,
  "error": false,
  "message": "success"
}
```

`data` is the save response row: `id` (uuid), `item_type`, `is_deleted` (boolean — `false`), `deleted_at` (null), `version` (integer), `enabled` (boolean — `false`), `parent_item_id` (uuid \| null), `name`, `description` (string \| null), `slug`, `full_path`, `depth` (integer), `cfg_data` (object, stored form), `created_at`, `created_by` (uuid \| null), `modified_at`, `modified_by` (uuid \| null). `meta` = `{ action: "update", id, slug, version, item_type }`.

**Errors**

| Status | Trigger                                                           | `message`                                                                                                                 | Code location                                                         |
| ------ | ----------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------- |
| 400    | Body field fails its format rule (as §8.5), `id` path param empty | issue message or `Schema validation error`                                                                                | `details[].code` (§11.2)                                              |
| 400    | Malformed JSON body                                               | the JSON parser's text                                                                                                    | no `details`                                                          |
| 401    | §3                                                                | per §3                                                                                                                    | —                                                                     |
| 403    | Caller lacks `delete` on `domain`                                 | `{ "error": "Authorization failed. You don't have access to perform this action." }`                                      | not returned                                                          |
| 404    | No item with this id                                              | `Config item not found`                                                                                                   | not returned                                                          |
| 404    | Id belongs to another item type                                   | `Config item type mismatch: expected domain, got <type> not found`                                                        | not returned                                                          |
| 500    | `id` not a uuid                                                   | `Failed to fetch config item`                                                                                             | not returned                                                          |
| 500    | `data.slug` differs and the current slug is published             | rename-blocked message ending `Code: IMMUTABLE_SLUG`                                                                      | inside `message` (suffix); `meta` = `{ action, id, slug, item_type }` |
| 500    | Validation of the resulting record failed                         | `Validation failed` (or, if the validator itself errors, the underlying error text with a single `INVALID` entry — §11.3) | `details[].code` (§11.3); `meta` as above                             |
| 500    | `data.full_path` empty                                            | storage not-null violation message naming the column                                                                      | not returned; `meta` as above                                         |

**Example**

```bash
curl -X DELETE "https://pivotly.example.com/api/v3/config-items/domain/5b0e8f3a-2c4d-4e6f-8a1b-3c5d7e9f1a2b" \
  -H "Authorization: Bearer $TOKEN" -H "X-Tenant-Id: $TENANT_ID"
```

**Notes:** to remove a published domain's storage, use the platform's permanent-delete operation (not covered by this document).

## 9. Endpoint reference — Publish

**Publication record.** Each publish run creates one publication record. §9.2–§9.4 return records in this shape (camelCase keys), restated per endpoint:

| Field                                                                                                                                          | Type                           | Notes                                                                                                                                                           |
| ---------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `id`                                                                                                                                           | string (uuid)                  | Publication id.                                                                                                                                                 |
| `domain`                                                                                                                                       | string                         | Domain slug (lower-cased and trimmed).                                                                                                                          |
| `createdBy`                                                                                                                                    | string (uuid) \| null          | Publishing user.                                                                                                                                                |
| `createdAt`, `modifiedAt`                                                                                                                      | string (ISO timestamp) \| null |                                                                                                                                                                 |
| `version`                                                                                                                                      | integer                        | The config item's version that was published.                                                                                                                   |
| `enabled`                                                                                                                                      | boolean                        | Always `true` (never written by publish).                                                                                                                       |
| `domainCfg`                                                                                                                                    | object                         | Snapshot of the published `cfg_data` (stored form), with `version` and `enabled` added by the platform.                                                         |
| `archiveEnabled`, `xrefEnabled`                                                                                                                | boolean                        | The run's archive / cross-reference flags (an absent `cfg_data.archive` / `xref_enabled` is recorded as `true`).                                                |
| `publishPhase`                                                                                                                                 | string                         | One of `not_set`, `not_started`, `pending_create`, `create`, `pending_prune`, `prune`, `pending_archive_update`, `archive_update`, `complete`.                  |
| `status`                                                                                                                                       | string                         | One of `not_set`, `staged`, `preprocessing`, `pending`, `processing`, `complete`, `cancel_requested`. `complete` means the run ended — successfully **or not**. |
| `createResult`, `pruneResult`, `archiveResult`                                                                                                 | string \| null                 | Step results: one of `not_set`, `success`, `warning`, `error`, `skipped`, `cleared`, `canceled`, `not_found`; `null` when the step has not run.                 |
| `createMessage`, `pruneMessage`, `archiveMessage`                                                                                              | string \| null                 | Step messages.                                                                                                                                                  |
| `createStartedAt`, `createEndedAt`, `pruneStartedAt`, `pruneEndedAt`, `archiveStartedAt`, `archiveEndedAt`                                     | string (ISO timestamp) \| null | Step timings.                                                                                                                                                   |
| `createResponse`, `createChanges`, `createLog`, `pruneResponse`, `pruneChanges`, `pruneLog`, `archiveResponse`, `archiveChanges`, `archiveLog` | object \| array \| null        | Step reports. ⚠️ Their inner structure is not a documented contract (not determinable from source as stable fields); treat as opaque diagnostics.               |

### 9.1 Publish a domain — POST /v3/domain-publish/{slug}

- **Classification:** consumer.
- **Headers:** `Authorization: Bearer <JWT>` (!), `X-Tenant-Id` (?), `X-App-Slug` (?).
- **Applicable global rules:** §2; §3 (verb `publish`), 403 body shape; §4; §6 (the published definition is the stored one, validated at save); §11.
- **Check order:** `[authz]` **first** → `[REST]` path → `[DB-publish]` steps.

**Path parameters**

| Param  | Type   | Req. | Bounds / format                                                                  |
| ------ | ------ | ---- | -------------------------------------------------------------------------------- |
| `slug` | string | !    | Length ≥ 1 `[REST]`; no pattern enforced; exact match against the domain's slug. |

**Request body:** none. Any body is ignored (a body that is not valid JSON is still rejected with 400, §2).

**Effective request contract**

- **Required to send:** path `slug`. **Optional:** nothing. **Ignored:** any body.
- **Publishes the stored definition:** the latest saved version of the domain with this slug (§8.5 / §8.6) — there is nothing to send, and the save-time validator is **not** re-run.
- **Not gated by `enabled`:** a soft-deleted (disabled) domain (§8.7) is still published.
- **No in-flight guard across runs:** a new run is recorded on every call, even while another run for the same slug is in progress. ⚠️ The outcome of overlapping runs is not determinable from source.

**Signature**

`publish(path slug!) — no body; publishes the latest saved version; runs create → prune → archive; records a publication`

**Process** `[DB-publish]`

1. **Load** the latest saved version of the domain (`item_type = domain`). None → 500 (see errors).
2. **Enqueue** — record a publication (`status: pending`, `publishPhase: pending_create`, `createResult: not_set`).
3. **Create** — create or alter the domain's storage and refresh its entry in the published-domain registry (§10.1). This step's report is stored on the publication.
4. **Prune** — then **Archive** — each recorded on the publication.
5. `warning` from any step still continues; the run then reports `warning` overall.

**Success — 200**

```json
{
  "data": {
    "response": {
      "result": "success",
      "message": "Domain publish complete",
      "error_message": null,
      "error_code": null,
      "error_detail": null,
      "meta": {
        "pub_id": "e1f2a3b4-c5d6-4e7f-8091-a2b3c4d5e6f7",
        "slug": "customer",
        "create_result": "success",
        "prune_result": "success",
        "archive result": "success"
      },
      "data": { "publish": {}, "prune": {}, "archive": {} }
    },
    "pub_id": "",
    "slug": "customer",
    "statuses": { "publish": "", "prune": "", "archive": "" },
    "step_responses": { "publish": {}, "prune": {}, "archive": {} },
    "has_warnings": false
  },
  "meta": {
    "pub_id": "e1f2a3b4-c5d6-4e7f-8091-a2b3c4d5e6f7",
    "slug": "customer",
    "create_result": "success",
    "prune_result": "success",
    "archive result": "success"
  },
  "status": 200,
  "error": false,
  "message": "Domain publish complete"
}
```

| Field                 | Type    | Notes                                                                                                                                                                                                                                                  |
| --------------------- | ------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `data.response`       | object  | The platform's run report: `result` (`success` \| `warning`), `message`, `error_message` (string \| null), `error_code` (string \| null), `error_detail` (string \| null), `meta`, `data`. **Read this object for the outcome** (see the table below). |
| `data.response.meta`  | object  | `pub_id` (uuid of the publication, §9.3), `slug`, `create_result`, `prune_result` and `archive result` (note: the last key contains a space) — each a step result value (§7).                                                                          |
| `data.response.data`  | object  | `{ publish, prune, archive }` — the three step reports. ⚠️ Inner structure not a documented contract.                                                                                                                                                  |
| `data.pub_id`         | string  | **Always `""`** — use `meta.pub_id` / `data.response.meta.pub_id` instead.                                                                                                                                                                             |
| `data.slug`           | string  | The path `slug`.                                                                                                                                                                                                                                       |
| `data.statuses`       | object  | `{ publish, prune, archive }` — **always `""`**; use `meta.create_result` / `prune_result` / `archive result` instead.                                                                                                                                 |
| `data.step_responses` | object  | `data.response.data` (the three step reports) when present; otherwise the placeholder `{ "publish": {}, "prune": {}, "archive": {} }`.                                                                                                                 |
| `data.has_warnings`   | boolean | **Always `false`** — check `data.response.result === "warning"` instead.                                                                                                                                                                               |
| `meta`                | object  | Same as `data.response.meta`.                                                                                                                                                                                                                          |
| `message`             | string  | Same as `data.response.message`.                                                                                                                                                                                                                       |

**How to read a 200** (all of these return HTTP 200):

| `data.response` looks like                                                                                                                                                                                                                 | Meaning                                                                                                                                                                                             |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `result: "success"`, `message: "Domain publish complete"`                                                                                                                                                                                  | Every step succeeded.                                                                                                                                                                               |
| `result: "warning"`, `message: "Domain publish complete"`                                                                                                                                                                                  | Completed; at least one step reported `warning` (see `meta` step results).                                                                                                                          |
| `result: "success"` or `"warning"` with `message: "Unhandled exception in domain publish process"` and a non-null `error_code` / `error_message`                                                                                           | **An unexpected error interrupted the run** after the create step started. Treat as a failure; inspect the publication (§9.3).                                                                      |
| **No `result` key** — `data.response` is the stored domain item wrapped in internal input keys rather than a run report; the top-level `meta` is `{}`, there is no top-level `message`, and `data.step_responses` is the empty placeholder | **The create step failed** in a way that was not converted to an error response. Treat as a failure; the publication (§9.4, latest for the slug) has `createResult: "error"` and a `createMessage`. |

**Errors**

| Status | Trigger                                                                | `message`                                                                                                                              | Code location                                                                                                                                                                       |
| ------ | ---------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 400    | `slug` empty                                                           | `Slug is required`                                                                                                                     | `details[].code` = `too_small`                                                                                                                                                      |
| 401    | §3                                                                     | per §3                                                                                                                                 | —                                                                                                                                                                                   |
| 403    | Caller lacks `publish` on `domain`                                     | `{ "error": "Authorization failed. You don't have access to perform this action." }`                                                   | not returned                                                                                                                                                                        |
| 500    | No domain config item with this slug                                   | the platform's lookup-failure text (it names internal storage; match on status + the absence of a publication rather than on the text) | not returned (`CFG_NOT_FOUND`); `meta` = `{ slug }`                                                                                                                                 |
| 500    | The publication could not be recorded                                  | the underlying reason text (e.g. `data.cfg_data must be a JSON object`)                                                                | not returned; `meta` = `{ process, err_code, err_msg, err_detail }` — `process` is an internal routine label and `err_code` a database error code; treat both as opaque diagnostics |
| 500    | The publication was recorded without an id                             | `Missing publication id from enqueue response`                                                                                         | not returned; `meta` = `{ step: "enqueue", enqueue_response }`                                                                                                                      |
| 500    | Create step finished with a result other than `success` / `warning`    | `Create step failed`                                                                                                                   | not returned; `meta` = `{ pub_id, slug, step: "publish", publish_status }`                                                                                                          |
| 500    | Prune step finished with a result other than `success` / `warning`     | `Prune step failed`                                                                                                                    | not returned; `meta` = `{ pub_id, slug, step: "prune", prune_result }`                                                                                                              |
| 500    | Archive step finished with a result other than `success` / `warning`   | `Archive step failed`                                                                                                                  | not returned; `meta` = `{ pub_id, slug, step: "archive", "archive result" }`                                                                                                        |
| 500    | Unexpected error in the prune or archive stage                         | the underlying reason text                                                                                                             | not returned; `meta` = `{ slug }`                                                                                                                                                   |
| 500    | Could not start the run (unexpected error while loading or enqueueing) | the underlying reason text                                                                                                             | not returned; `meta` = `{ slug }`                                                                                                                                                   |

On every 500 above the publication record (when one was created) remains and shows the failed step (§9.3, §9.4). Failures are **not** returned as 4xx, and the step failures carry no `details`.

**Example**

```bash
curl -X POST "https://pivotly.example.com/api/v3/domain-publish/customer" \
  -H "Authorization: Bearer $TOKEN" -H "X-Tenant-Id: $TENANT_ID"
```

**Notes:** not idempotent in its records — every call creates a publication. After a successful publish, §8.4 reports `is_published: true` and §8.3 `publish_status: "complete"` until the next save.

### 9.2 List successful publications — GET /v3/domain-publications/published

- **Classification:** consumer.
- **Purpose:** every publication record whose `createResult` is exactly `success`, for all domains and versions (runs whose create step ended in `warning` are **not** included). Unordered; not paginated.
- **Headers:** `Authorization: Bearer <JWT>` (!), `X-Tenant-Id` (?), `X-App-Slug` (?).
- **Applicable global rules:** §2; §3 (verb `view`), 403 body shape; §4.
- **Check order:** `[authz]` only (no parameters).

**Parameters:** none.

**Success — 200**: `data` is an array of publication records — each `id` (uuid), `domain` (string), `createdBy` (uuid \| null), `createdAt` / `modifiedAt` (ISO timestamp \| null), `version` (integer), `enabled` (boolean, always `true`), `domainCfg` (object), `archiveEnabled` / `xrefEnabled` (boolean), `publishPhase` (string, §7), `status` (string, §7), `createResult` / `pruneResult` / `archiveResult` (string \| null, §7), `createMessage` / `pruneMessage` / `archiveMessage` (string \| null), the six step timestamps (ISO timestamp \| null), and the nine step reports (opaque, ⚠️). No `message`.

```json
{
  "data": [
    {
      "id": "e1f2a3b4-c5d6-4e7f-8091-a2b3c4d5e6f7",
      "domain": "customer",
      "createdBy": "7d2b9a10-4c3e-4f5a-9b8c-1d2e3f4a5b6c",
      "createdAt": "2026-09-28T02:10:00.512Z",
      "modifiedAt": "2026-09-28T02:10:03.207Z",
      "version": 2,
      "enabled": true,
      "domainCfg": {
        "domain_table_name": "customer",
        "version": 2,
        "enabled": true
      },
      "archiveEnabled": true,
      "xrefEnabled": true,
      "publishPhase": "complete",
      "status": "complete",
      "createResult": "success",
      "createMessage": "OK",
      "createStartedAt": "2026-09-28T02:10:00.620Z",
      "createEndedAt": "2026-09-28T02:10:02.114Z",
      "createResponse": {},
      "createChanges": {},
      "createLog": {},
      "pruneResult": "success",
      "pruneMessage": "OK",
      "pruneStartedAt": "2026-09-28T02:10:02.120Z",
      "pruneEndedAt": "2026-09-28T02:10:02.801Z",
      "pruneResponse": {},
      "pruneChanges": {},
      "pruneLog": {},
      "archiveResult": "success",
      "archiveMessage": "OK",
      "archiveStartedAt": "2026-09-28T02:10:02.805Z",
      "archiveEndedAt": "2026-09-28T02:10:03.200Z",
      "archiveResponse": {},
      "archiveChanges": {},
      "archiveLog": {}
    }
  ],
  "meta": {},
  "status": 200,
  "error": false
}
```

(`domainCfg` abbreviated for readability — it carries the full published configuration.)

**Errors**

| Status | Trigger                         | `message` / body                                                                     | Code location |
| ------ | ------------------------------- | ------------------------------------------------------------------------------------ | ------------- |
| 401    | §3                              | per §3                                                                               | —             |
| 403    | Caller lacks `view` on `domain` | `{ "error": "Authorization failed. You don't have access to perform this action." }` | not returned  |
| 500    | Query failed                    | `Failed to fetch published domains`                                                  | not returned  |

**Example**

```bash
curl "https://pivotly.example.com/api/v3/domain-publications/published" \
  -H "Authorization: Bearer $TOKEN" -H "X-Tenant-Id: $TENANT_ID"
```

**Notes:** returns every successful run, not one row per domain; the full list can be large. For "which domains are live now", use §10.1.

### 9.3 Get a publication by id — GET /v3/domain-publications/{id}

- **Classification:** consumer.
- **Headers:** `Authorization: Bearer <JWT>` (!), `X-Tenant-Id` (?), `X-App-Slug` (?).
- **Applicable global rules:** §2; §3 (verb `view`), 403 body shape; §4.
- **Check order:** `[REST]` params → `[authz]`.

**Path parameters**

| Param | Type   | Req. | Bounds / format                                                                                                                                     |
| ----- | ------ | ---- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| `id`  | string | !    | No length or pattern check at the request layer. Must be a full uuid — any other value makes the lookup fail with 500. Use `meta.pub_id` from §9.1. |

**Success — 200**: `data` is one publication record — `id` (uuid), `domain` (string), `createdBy` (uuid \| null), `createdAt` / `modifiedAt` (ISO timestamp \| null), `version` (integer), `enabled` (boolean, always `true`), `domainCfg` (object), `archiveEnabled` / `xrefEnabled` (boolean), `publishPhase` (string, §7), `status` (string, §7), `createResult` / `pruneResult` / `archiveResult` (string \| null, §7), `createMessage` / `pruneMessage` / `archiveMessage` (string \| null), the six step timestamps (ISO timestamp \| null), and the nine step reports (opaque, ⚠️). No `message`.

```json
{
  "data": {
    "id": "e1f2a3b4-c5d6-4e7f-8091-a2b3c4d5e6f7",
    "domain": "customer",
    "createdBy": "7d2b9a10-4c3e-4f5a-9b8c-1d2e3f4a5b6c",
    "createdAt": "2026-09-28T02:10:00.512Z",
    "modifiedAt": "2026-09-28T02:10:02.114Z",
    "version": 2,
    "enabled": true,
    "domainCfg": {
      "domain_table_name": "customer",
      "version": 2,
      "enabled": true
    },
    "archiveEnabled": true,
    "xrefEnabled": true,
    "publishPhase": "create",
    "status": "complete",
    "createResult": "error",
    "createMessage": "Create failed. err_msg: ...",
    "createStartedAt": "2026-09-28T02:10:00.620Z",
    "createEndedAt": "2026-09-28T02:10:02.114Z",
    "createResponse": null,
    "createChanges": null,
    "createLog": {},
    "pruneResult": null,
    "pruneMessage": null,
    "pruneStartedAt": null,
    "pruneEndedAt": null,
    "pruneResponse": null,
    "pruneChanges": null,
    "pruneLog": null,
    "archiveResult": null,
    "archiveMessage": null,
    "archiveStartedAt": null,
    "archiveEndedAt": null,
    "archiveResponse": null,
    "archiveChanges": null,
    "archiveLog": null
  },
  "meta": {},
  "status": 200,
  "error": false
}
```

(A failed run, shown for contrast; `domainCfg` abbreviated.)

**Errors**

| Status | Trigger                           | `message` / body                                                                     | Code location |
| ------ | --------------------------------- | ------------------------------------------------------------------------------------ | ------------- |
| 401    | §3                                | per §3                                                                               | —             |
| 403    | Caller lacks `view` on `domain`   | `{ "error": "Authorization failed. You don't have access to perform this action." }` | not returned  |
| 404    | No publication with this id       | `Domain publication not found`                                                       | not returned  |
| 500    | `id` not a uuid, or lookup failed | `Failed to fetch domain publication`                                                 | not returned  |

**Example**

```bash
curl "https://pivotly.example.com/api/v3/domain-publications/e1f2a3b4-c5d6-4e7f-8091-a2b3c4d5e6f7" \
  -H "Authorization: Bearer $TOKEN" -H "X-Tenant-Id: $TENANT_ID"
```

### 9.4 List publications of a domain — GET /v3/domain-publications/domain/{domain}

- **Classification:** consumer.
- **Purpose:** every publication record of one domain (all runs, all results), newest version first. Not paginated.
- **Headers:** `Authorization: Bearer <JWT>` (!), `X-Tenant-Id` (?), `X-App-Slug` (?).
- **Applicable global rules:** §2; §3 (verb `view`), 403 body shape; §4.
- **Check order:** `[REST]` params → `[authz]`.

**Path parameters**

| Param    | Type   | Req. | Bounds / format                                                                                                                                                                             |
| -------- | ------ | ---- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `domain` | string | !    | Length ≥ 1 `[REST]` (`Domain is required`); no pattern enforced; exact, case-sensitive match against the lower-cased slug recorded on the publication. An unknown value returns `[]` (200). |

**Success — 200**: `data` is an array of publication records, ordered by `version` descending (⚠️ the order of several runs of the same version is not determinable from source) — each `id` (uuid), `domain` (string), `createdBy` (uuid \| null), `createdAt` / `modifiedAt` (ISO timestamp \| null), `version` (integer), `enabled` (boolean, always `true`), `domainCfg` (object), `archiveEnabled` / `xrefEnabled` (boolean), `publishPhase` (string, §7), `status` (string, §7), `createResult` / `pruneResult` / `archiveResult` (string \| null, §7), `createMessage` / `pruneMessage` / `archiveMessage` (string \| null), the six step timestamps (ISO timestamp \| null), and the nine step reports (opaque, ⚠️). No `message`.

```json
{
  "data": [
    {
      "id": "e1f2a3b4-c5d6-4e7f-8091-a2b3c4d5e6f7",
      "domain": "customer",
      "createdBy": "7d2b9a10-4c3e-4f5a-9b8c-1d2e3f4a5b6c",
      "createdAt": "2026-09-28T02:10:00.512Z",
      "modifiedAt": "2026-09-28T02:10:03.207Z",
      "version": 2,
      "enabled": true,
      "domainCfg": {
        "domain_table_name": "customer",
        "version": 2,
        "enabled": true
      },
      "archiveEnabled": true,
      "xrefEnabled": true,
      "publishPhase": "complete",
      "status": "complete",
      "createResult": "success",
      "createMessage": "OK",
      "createStartedAt": "2026-09-28T02:10:00.620Z",
      "createEndedAt": "2026-09-28T02:10:02.114Z",
      "createResponse": {},
      "createChanges": {},
      "createLog": {},
      "pruneResult": "success",
      "pruneMessage": "OK",
      "pruneStartedAt": "2026-09-28T02:10:02.120Z",
      "pruneEndedAt": "2026-09-28T02:10:02.801Z",
      "pruneResponse": {},
      "pruneChanges": {},
      "pruneLog": {},
      "archiveResult": "success",
      "archiveMessage": "OK",
      "archiveStartedAt": "2026-09-28T02:10:02.805Z",
      "archiveEndedAt": "2026-09-28T02:10:03.200Z",
      "archiveResponse": {},
      "archiveChanges": {},
      "archiveLog": {}
    }
  ],
  "meta": {},
  "status": 200,
  "error": false
}
```

(`domainCfg` abbreviated.)

**Errors**

| Status | Trigger                         | `message` / body                                                                     | Code location                  |
| ------ | ------------------------------- | ------------------------------------------------------------------------------------ | ------------------------------ |
| 400    | `domain` empty                  | `Domain is required`                                                                 | `details[].code` = `too_small` |
| 401    | §3                              | per §3                                                                               | —                              |
| 403    | Caller lacks `view` on `domain` | `{ "error": "Authorization failed. You don't have access to perform this action." }` | not returned                   |
| 500    | Query failed                    | `Failed to fetch domain publications by domain`                                      | not returned                   |

**Example**

```bash
curl "https://pivotly.example.com/api/v3/domain-publications/domain/customer" \
  -H "Authorization: Bearer $TOKEN" -H "X-Tenant-Id: $TENANT_ID"
```

**Notes:** the first row is the most recent version's run; check its `status` (`complete` = finished) and `createResult` / `pruneResult` / `archiveResult` to know whether that run succeeded.

## 10. Endpoint reference — Type-specific operations

### 10.1 List published domains — GET /v3/domain-info

- **Classification:** consumer.
- **Purpose:** one entry per domain currently in the published-domain registry, with its runtime metadata, sorted by slug. Includes platform system datasets (slugs beginning `sys_`) registered by the platform.
- **Headers:** `Authorization: Bearer <JWT>` (!), `X-Tenant-Id` (?), `X-App-Slug` (?).
- **Applicable global rules:** §2; §3 (verb `list`), 403 body shape; §4.
- **Check order:** `[REST]` query → `[authz]`.

**Query parameters**

| Param      | Type              | Req. | Bounds / format | Default              | Notes                                                                                                                          |
| ---------- | ----------------- | ---- | --------------- | -------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| `page`     | integer (coerced) | ?    | ≥ 0             | `0` when paginating  | Zero-based.                                                                                                                    |
| `pageSize` | integer (coerced) | ?    | 1–100           | `25` when paginating |                                                                                                                                |
| `search`   | string            | ?    | None enforced   | —                    | Trimmed, then case-insensitive substring match on the slug (`domain`). Blank = no filter. `%`/`_` are literal characters here. |

**Paging rule:** when **neither** `page` nor `pageSize` is sent, the full (filtered) list is returned and `pagination` = `{ page: 0, page_size: <number of entries>, total_records: <number of entries> }`. When either is sent, the missing one takes its default and one page is returned. Unknown query parameters are stripped.

**Success — 200**

```json
{
  "data": {
    "domainInfoList": [
      {
        "domain": "customer",
        "createdAt": "2026-09-20 02:30:11.482+00",
        "modifiedAt": "2026-09-28 02:10:02.101+00",
        "domainCfg": { "domain_table_name": "customer" },
        "version": 2,
        "enabled": true,
        "accessControlModel": "all_systems",
        "hardDeleteAllowed": true,
        "directWriteEnabled": true,
        "colsAll": ["customer_name", "email", "status"],
        "colTypes": {
          "customer_name": "text",
          "email": "text",
          "status": "text"
        },
        "notnullCols": ["customer_name"],
        "tvcCols": ["customer_name"],
        "fwwCols": [],
        "hasFwwCols": false,
        "fkResolverCfg": {},
        "fkResolverCols": [],
        "hasFkResolvers": false,
        "archiveEnabled": true,
        "archiveMaxRecordVersions": 10000,
        "archiveRetentionDays": 0,
        "xrefEnabled": true,
        "uniqueCols": ["email"],
        "metaDataCols": ["id", "version", "created_at", "modified_at"],
        "trackVersions": true,
        "multiTenant": false,
        "dedupeEnabled": false
      }
    ]
  },
  "meta": {},
  "status": 200,
  "error": false,
  "pagination": { "page": 0, "page_size": 1, "total_records": 1 }
}
```

`data.domainInfoList[]` entry fields:

| Field                                                                                          | Type                    | Notes                                                                                                                                                                                                                                                           |
| ---------------------------------------------------------------------------------------------- | ----------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `domain`                                                                                       | string                  | Slug.                                                                                                                                                                                                                                                           |
| `createdAt`, `modifiedAt`                                                                      | string \| null          | Timestamps as a database-formatted string (e.g. `2026-09-20 02:30:11.482+00`), **not** ISO-8601 with `T`.                                                                                                                                                       |
| `domainCfg`                                                                                    | object                  | The published configuration.                                                                                                                                                                                                                                    |
| `version`                                                                                      | integer                 | Published version.                                                                                                                                                                                                                                              |
| `enabled`                                                                                      | boolean                 |                                                                                                                                                                                                                                                                 |
| `accessControlModel`                                                                           | string \| null          | e.g. `all_systems` (the value injected at save, §6.3).                                                                                                                                                                                                          |
| `hardDeleteAllowed`, `directWriteEnabled`                                                      | boolean \| null         | From the injected access control.                                                                                                                                                                                                                               |
| `colsAll`, `notnullCols`, `tvcCols`, `fwwCols`, `fkResolverCols`, `uniqueCols`, `metaDataCols` | array of string \| null | Column lists: all columns, not-null columns, version-triggering columns, first-write-wins columns, FK-resolving columns, unique columns, platform meta columns. ⚠️ Exact membership rules are computed at publish and not determinable from the endpoints here. |
| `colTypes`, `fkResolverCfg`                                                                    | object \| null          | Column → type map; FK resolver configuration. ⚠️ Inner structure not a documented contract.                                                                                                                                                                     |
| `hasFwwCols`, `hasFkResolvers`                                                                 | boolean \| null         |                                                                                                                                                                                                                                                                 |
| `archiveEnabled`, `xrefEnabled`, `trackVersions`, `multiTenant`, `dedupeEnabled`               | boolean \| null         | Published capability flags.                                                                                                                                                                                                                                     |
| `archiveMaxRecordVersions`, `archiveRetentionDays`                                             | integer \| null         | Published archive policy.                                                                                                                                                                                                                                       |

(Example values are illustrative; `domainCfg` abbreviated.)

**Errors**

| Status | Trigger                                                      | `message` / body                                                                     | Code location            |
| ------ | ------------------------------------------------------------ | ------------------------------------------------------------------------------------ | ------------------------ |
| 400    | Non-numeric, non-integer or out-of-range `page` / `pageSize` | issue message or `Schema validation error`                                           | `details[].code` (§11.2) |
| 401    | §3                                                           | per §3                                                                               | —                        |
| 403    | Caller lacks `list` on `domain`                              | `{ "error": "Authorization failed. You don't have access to perform this action." }` | not returned             |
| 500    | Query failed                                                 | `Failed to fetch domain info`                                                        | not returned             |

**Example**

```bash
curl -G "https://pivotly.example.com/api/v3/domain-info" \
  -H "Authorization: Bearer $TOKEN" -H "X-Tenant-Id: $TENANT_ID" \
  --data-urlencode "search=cust" --data-urlencode "page=0" --data-urlencode "pageSize=25"
```

**Notes:** this is the registry the §6.9 immutability rules compare against: a slug listed here counts as published.

## 11. Error codes

### 11.1 Consolidated table

"Where it appears" says how a client can recognize the condition on the wire. **The error envelope has no code field**: most platform conditions are identified only by HTTP status plus `message`. The "Label" column gives the platform's name for a condition where it has one; a label marked _(not returned)_ never appears in the response.

| Label                                                           | HTTP            | Where it appears                                                                                                                                                                                                                                                                                                            | Endpoint(s)                                          | Meaning / trigger                                                                                                                            |
| --------------------------------------------------------------- | --------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| Request-shape validation                                        | 400             | `message` (single issue text, or `Schema validation error`); issue codes in `details[].code` (§11.2)                                                                                                                                                                                                                        | all with parameters or a body                        | A path, query or body field violates its request-layer rule (type, length, range, pattern, enum, required). See each endpoint's field table. |
| Malformed JSON body / unsupported content type / body too large | 400 / 415 / 413 | `message`                                                                                                                                                                                                                                                                                                                   | §8.5, §8.6, §8.7, §9.1                               | §2.                                                                                                                                          |
| Item type mismatch                                              | 400             | `message` `Item type mismatch: URL specifies "domain" but body specifies "<t>".`                                                                                                                                                                                                                                            | §8.5, §8.6                                           | `data.item_type` ≠ `domain`.                                                                                                                 |
| Id on create                                                    | 400             | `message` `ID should not be provided for creation. Use PATCH for updates.`                                                                                                                                                                                                                                                  | §8.5                                                 | `data.id` or `parameters.id` sent.                                                                                                           |
| Id mismatch                                                     | 400             | `message` `ID mismatch between URL and body.`                                                                                                                                                                                                                                                                               | §8.6                                                 | `data.id` / `parameters.id` ≠ path id.                                                                                                       |
| `USER_NOT_IN_IAM`                                               | 401             | `meta.code`                                                                                                                                                                                                                                                                                                                 | all                                                  | Valid token, but no registered platform user.                                                                                                |
| Authentication failure                                          | 401             | `message`                                                                                                                                                                                                                                                                                                                   | all                                                  | Missing/invalid bearer token (§3).                                                                                                           |
| Authorization — unresolved object type                          | 403             | non-envelope body `{ "error": "Authorization Failed. Incorrect config or invalid id was provided" }`                                                                                                                                                                                                                        | §8.1                                                 | `itemType` missing or empty.                                                                                                                 |
| Authorization — denied                                          | 403             | non-envelope body `{ "error": "Authorization failed. You don't have access to perform this action." }`                                                                                                                                                                                                                      | all                                                  | The caller's roles do not grant the endpoint's verb on `domain` (§3).                                                                        |
| Config item not found                                           | 404             | `message` `Config item not found`                                                                                                                                                                                                                                                                                           | §8.3, §8.7                                           | No item with the id.                                                                                                                         |
| Config item type mismatch                                       | 404             | `message` `Config item type mismatch: expected domain, got <type> not found`                                                                                                                                                                                                                                                | §8.3, §8.7                                           | The id belongs to another type.                                                                                                              |
| Slug not found                                                  | 404             | `message` `domain with slug "<slug>" not found`                                                                                                                                                                                                                                                                             | §8.2                                                 | No domain with the slug.                                                                                                                     |
| Publication not found                                           | 404             | `message` `Domain publication not found`                                                                                                                                                                                                                                                                                    | §9.3                                                 | No publication with the id.                                                                                                                  |
| `INVALID` _(not returned)_ — domain validation failed           | **500**         | `message` `Validation failed`; every violation in `details[]` with its code in `details[].code` (§11.3); `meta` = `{ action, id, slug, item_type }`. Exception: when the validator itself errors, `message` is the underlying error text and `details[]` holds a single `INVALID` entry                                     | §8.5, §8.6, §8.7                                     | One or more §6 save rules failed.                                                                                                            |
| Slug taken (same type) _(not returned)_                         | **500**         | `message` states the slug already exists and quotes the existing item's id                                                                                                                                                                                                                                                  | §8.5 (and §8.6 unknown id)                           | `data.slug` already used by a domain.                                                                                                        |
| Slug taken (other type) _(not returned)_                        | **500**         | `message` `Slug "<slug>" already exists with item_type "<type>"; cannot save as "domain".`                                                                                                                                                                                                                                  | §8.5 (and §8.6 unknown id)                           | `data.slug` used by another type.                                                                                                            |
| `IMMUTABLE_SLUG`                                                | **500**         | **suffix of `message`** (`... Code: IMMUTABLE_SLUG`)                                                                                                                                                                                                                                                                        | §8.6, §8.7                                           | Slug changed while the current slug is in the published-domain registry (§6.9).                                                              |
| Item type changed _(not returned)_                              | **500**         | `message` `Item type conflict for slug "<slug>" (existing="<t>", provided="domain")`                                                                                                                                                                                                                                        | §8.6                                                 | The id belongs to another type.                                                                                                              |
| Storage constraint violation _(not returned)_                   | **500**         | `message` = the storage error text                                                                                                                                                                                                                                                                                          | §8.5, §8.6, §8.7                                     | Slug fails the storage pattern (e.g. starts with a digit); renamed slug already used; `enabled: null` or empty `full_path` on update.        |
| Non-uuid id on update _(not returned)_                          | **500**         | `message` `Internal server error` (production)                                                                                                                                                                                                                                                                              | §8.6                                                 | Path `id` not a uuid.                                                                                                                        |
| Lookup/query failure _(not returned)_                           | **500**         | `message` `Failed to fetch config items by columns` / `Failed to fetch config item by slug` / `Failed to fetch config item` / `Failed to fetch config items` / `Failed to fetch published domains` / `Failed to fetch domain publication` / `Failed to fetch domain publications by domain` / `Failed to fetch domain info` | §8.1, §8.2, §8.3/§8.7, §8.4, §9.2, §9.3, §9.4, §10.1 | Invalid filter value for the field's type, non-uuid id, or a platform failure.                                                               |
| `CFG_NOT_FOUND` _(not returned)_                                | **500**         | `message` = platform lookup-failure text; `meta.slug`                                                                                                                                                                                                                                                                       | §9.1                                                 | No domain config item with the slug.                                                                                                         |
| Publish could not start / enqueue failed _(not returned)_       | **500**         | `message` = underlying reason; `meta` = diagnostics                                                                                                                                                                                                                                                                         | §9.1                                                 | Unexpected error while loading or recording the run.                                                                                         |
| Create / Prune / Archive step failed _(not returned)_           | **500**         | `message` `Create step failed` / `Prune step failed` / `Archive step failed`; `meta` = `{ pub_id, slug, step, <step result> }`                                                                                                                                                                                              | §9.1                                                 | A step finished with a result other than `success` / `warning`.                                                                              |
| Unhandled prune / archive error _(not returned)_                | **500**         | `message` = underlying reason; `meta.slug`                                                                                                                                                                                                                                                                                  | §9.1                                                 | Unexpected error in those stages.                                                                                                            |
| Publish failure reported as 200                                 | **200**         | `data.response` lacks `result`, or has `message` `Unhandled exception in domain publish process` with a non-null `error_code`                                                                                                                                                                                               | §9.1                                                 | See §9.1 "How to read a 200".                                                                                                                |
| Authorization check failure                                     | **500**         | `message` (generic)                                                                                                                                                                                                                                                                                                         | all                                                  | The permission check itself failed (e.g. `X-Tenant-Id` not a uuid).                                                                          |

**4xx vs 5xx.** Only request-shape problems, the API-side type/id checks, authentication, authorization and not-found conditions return 4xx. **Every platform-side validation and state rejection** — duplicate slug, slug rename after publish, type change, domain validation, storage constraints and every publish failure — currently surfaces as **500** carrying a descriptive `message` (and, for save validation, `details[]`), and some publish failures surface as **200** (§9.1). Clients must treat a 500 with `details[]` or a recognizable `message` as a caller error, not a transient fault; do not retry such requests unchanged.

### 11.2 Request-layer issue codes (`details[].code` on 400)

Each `details[]` entry is `{ field: string, message: string, code: string }`, where `field` is the dotted path of the offending value (e.g. `data.slug`, `data.cfg_data`, `pageSize`, `domain`) and `message` is the rule's text. Codes the endpoints in this document can produce:

| Code             | Trigger                                                                                                                                                                                       |
| ---------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `invalid_type`   | Missing required field, `null` where not allowed, wrong JSON type (e.g. `cfg_data` an array or string, `name: null`, `enabled: "yes"`), or a non-numeric value for a numeric query parameter. |
| `too_small`      | Below a minimum: empty `slug` / `item_type` / path `slug` / path `id` / path `domain`; `page` below its minimum; `pageSize` < 1; `version` ≤ 0.                                               |
| `too_big`        | Above a maximum: `slug` > 255, `item_type` > 100, `name` > 255, `description` > 1000, `full_path` > 500, `pageSize` > 100, `limit` > 100.                                                     |
| `invalid_format` | `id`, `parameters.id` or `parent_item_id` not matching the uuid pattern (`Invalid UUID format`).                                                                                              |
| `invalid_value`  | `fetchType` not `list` / `paginated`.                                                                                                                                                         |

A non-integer value for an integer parameter (e.g. `page=1.5`) is rejected with `invalid_type` (expected an integer).

### 11.3 Domain validation sub-codes (`details[].code` on the 500 `Validation failed`)

Each entry is `{ path, code, detail, severity, remediation, current?, proposed? }`: `path` (string; JSON-path style such as `$.cfg_data.schema[2].name`, except `TRIGGERS_VERSION_CHANGE_REQUIRED`, which uses `/cfg_data/schema/<n>/triggers_version_change`), `code`, `detail` (string), `severity` (always `error` in returned entries), `remediation` (`none` \| `rename_and_retry` \| `drop_and_recreate`), `current` / `proposed` (included where shown). All violations are reported together.

| Code                                 | Path(s)                                                                                                                 | Triggering rule (exact)                                                                                                                                                                                                                                                         | Remediation                            | Defined in |
| ------------------------------------ | ----------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------- | ---------- |
| `REQUIRED`                           | `$.name` (reachable); also defined for `$.slug`, `$.item_type`, `$.version`, which cannot occur through these endpoints | The resolved `name` is empty — on update / soft-delete with an empty or whitespace-only `name`                                                                                                                                                                                  | `none`                                 | §6.1       |
| `REQUIRED`                           | `$.cfg_data.domain_name_singular`, `$.cfg_data.default_conflict_resolution`, `$.cfg_data.schema`                        | Absent (or `""` for `domain_name_singular`)                                                                                                                                                                                                                                     | `none`                                 | §6.2       |
| `REQUIRED`                           | `...schema[<n>].name`, `.column_name`, `.data_type`                                                                     | Absent or `""`                                                                                                                                                                                                                                                                  | `none`                                 | §6.5       |
| `REQUIRED`                           | `...schema[<n>].masking.masked`                                                                                         | `masking` is an object and `masked` is absent or not a JSON boolean                                                                                                                                                                                                             | `none`                                 | §6.5       |
| `REQUIRED`                           | `$.cfg_data.system_access[<n>].system_slug`, `...change_tracking.mode`                                                  | Absent or `""`                                                                                                                                                                                                                                                                  | `none`                                 | §6.7       |
| `REQUIRED`                           | `$.cfg_data.indexes[<n>].columns`                                                                                       | Absent, not an array, or empty                                                                                                                                                                                                                                                  | `none`                                 | §6.8       |
| `TYPE`                               | `$.cfg_data`                                                                                                            | Not a JSON object (no other `cfg_data` rule then runs)                                                                                                                                                                                                                          | `none`                                 | §6.1       |
| `TYPE`                               | `$.cfg_data.domain_name_singular`                                                                                       | Not a JSON string                                                                                                                                                                                                                                                               | `none`                                 | §6.2       |
| `TYPE`                               | `$.cfg_data.default_conflict_resolution`                                                                                | Not an object                                                                                                                                                                                                                                                                   | `none`                                 | §6.2       |
| `TYPE`                               | `$.cfg_data.area`                                                                                                       | Not a string or array; or an array with a non-string element                                                                                                                                                                                                                    | `none`                                 | §6.2       |
| `TYPE`                               | `$.cfg_data.archive`, `.xref_enabled`, `.track_versions`, `.multi_tenant`, `.dedupe_enabled`                            | Key present and value not a JSON boolean (`null` included)                                                                                                                                                                                                                      | `none`                                 | §6.2       |
| `TYPE`                               | `$.cfg_data.schema`, `$.cfg_data.schema[<n>]`                                                                           | `schema` not an array; an entry not an object                                                                                                                                                                                                                                   | `none`                                 | §6.2, §6.5 |
| `TYPE`                               | `...schema[<n>].masking`, `.conflict_resolution`, `.fk_config`                                                          | Present and not an object                                                                                                                                                                                                                                                       | `none`                                 | §6.5       |
| `TYPE`                               | `$.cfg_data.system_access`, `...system_access[<n>]`, `...change_tracking`                                               | Not an array; entry not an object; `change_tracking` not an object                                                                                                                                                                                                              | `none`                                 | §6.7       |
| `TYPE`                               | `$.cfg_data.indexes`, `$.cfg_data.indexes[<n>]`                                                                         | Present and not an array (`null` included); entry not an object                                                                                                                                                                                                                 | `none`                                 | §6.8       |
| `VALUE`                              | `$.slug`                                                                                                                | Longer than 50 chars (`current`); or not matching `^[a-z0-9](?:[a-z0-9_]*[a-z0-9])?$` (`current`) — two separate entries                                                                                                                                                        | `rename_and_retry`                     | §6.1       |
| `VALUE`                              | `$.cfg_data.default_conflict_resolution.code`                                                                           | Absent or not `lww` / `fww`                                                                                                                                                                                                                                                     | `none`                                 | §6.4       |
| `VALUE`                              | `$.cfg_data.archive_policy.max_record_versions`, `.retention_days`                                                      | Present and not a whole JSON number ≥ 0                                                                                                                                                                                                                                         | `none`                                 | §6.4       |
| `VALUE`                              | `$.cfg_data.schema`                                                                                                     | Empty array                                                                                                                                                                                                                                                                     | `none`                                 | §6.2       |
| `VALUE`                              | `...schema[<n>].name`, `.column_name`                                                                                   | Not matching `^(?!_)[A-Za-z0-9_]+$`; `column_name` > 63 chars                                                                                                                                                                                                                   | `rename_and_retry`                     | §6.5       |
| `VALUE`                              | `...schema[<n>].data_type`                                                                                              | Not one of the 37 types (§7)                                                                                                                                                                                                                                                    | `none`                                 | §6.5       |
| `VALUE`                              | `...schema[<n>].conflict_resolution.code`                                                                               | Present and not `lww` / `fww`                                                                                                                                                                                                                                                   | `none`                                 | §6.5       |
| `VALUE`                              | `...schema[<n>].picklist.slug`                                                                                          | Not matching `^[a-z][a-z0-9_-]{0,127}$`                                                                                                                                                                                                                                         | `rename_and_retry`                     | §6.5       |
| `VALUE`                              | `...change_tracking.mode`                                                                                               | Not `specific_columns`                                                                                                                                                                                                                                                          | `none`                                 | §6.7       |
| `VALUE`                              | `$.cfg_data.indexes[<n>].name`                                                                                          | > 50 chars; or not matching `^[a-z0-9](?:[a-z0-9_]*[a-z0-9])?$`                                                                                                                                                                                                                 | `rename_and_retry`                     | §6.8       |
| `INVALID`                            | `$`                                                                                                                     | The validator could not evaluate the payload (e.g. a `unique` value that cannot be read as a boolean). In this case the 500's `message` is the underlying error text (**not** `Validation failed`) and `details[]` holds only this one entry — all other violations are dropped | `none`                                 | §6.5, §6.8 |
| `RESERVED_PREFIX_SLUG`               | `$.slug`                                                                                                                | Slug starts with `sys_`                                                                                                                                                                                                                                                         | `rename_and_retry`                     | §6.1       |
| `RESERVED_PREFIX_USDF`               | `$.slug`                                                                                                                | Slug starts with `dvw_`, `dtm_` or `dtv_`                                                                                                                                                                                                                                       | `rename_and_retry`                     | §6.1       |
| `SLUG_TABLE_MATCH`                   | `$.cfg_data.domain_table_name`                                                                                          | `domain_table_name` absent/`""`, or ≠ slug (`current`, `proposed` = slug)                                                                                                                                                                                                       | `none`                                 | §6.2       |
| `XREF_REQUIRES_TRACK`                | `$.cfg_data.xref_enabled`                                                                                               | `xref_enabled` is `true` and `track_versions` is not `true`                                                                                                                                                                                                                     | `none`                                 | §6.2       |
| `ARCHIVE_POLICY_REQUIRED`            | `$.cfg_data.archive_policy`                                                                                             | `archive` is `true` and `archive_policy` is absent or not an object                                                                                                                                                                                                             | `none`                                 | §6.2       |
| `INVALID_READ_VIEW_SLUG`             | `$.cfg_data.read_view_slug`                                                                                             | Present, not `null`, and not a non-blank string matching `^[a-z][a-z0-9_-]{0,127}$`                                                                                                                                                                                             | `rename_and_retry`                     | §6.2       |
| `RESERVED_ATTR_NAME`                 | `...schema[<n>].name`                                                                                                   | Lower-cased name is a reserved name (§7)                                                                                                                                                                                                                                        | `rename_and_retry`                     | §6.5       |
| `RESERVED_COL_NAME`                  | `...schema[<n>].column_name`                                                                                            | Lower-cased column name is a reserved name (§7)                                                                                                                                                                                                                                 | `rename_and_retry`                     | §6.5       |
| `RESERVED_PREFIX_SYS`                | `...schema[<n>].name`, `.column_name`                                                                                   | Starts with `sys_` (case-insensitive)                                                                                                                                                                                                                                           | `rename_and_retry`                     | §6.5       |
| `RESERVED_PREFIX_UNDERSCORE`         | `...schema[<n>].name`, `.column_name`                                                                                   | Starts with `_`                                                                                                                                                                                                                                                                 | `rename_and_retry`                     | §6.5       |
| `RESERVED_PREFIX_P`                  | `...schema[<n>].name`, `.column_name`                                                                                   | Starts with `p_` (case-insensitive)                                                                                                                                                                                                                                             | `rename_and_retry`                     | §6.5       |
| `JSON_DOC_REQUIRES_JSONB`            | `...schema[<n>].json_doc`                                                                                               | `json_doc` present and `data_type` present and ≠ `jsonb`                                                                                                                                                                                                                        | `none`                                 | §6.5       |
| `TRIGGERS_VERSION_CHANGE_REQUIRED`   | `/cfg_data/schema/<n>/triggers_version_change`                                                                          | Absent or not a JSON boolean (`proposed` = the sent value)                                                                                                                                                                                                                      | `none`                                 | §6.5       |
| `INVALID_DEFAULT_VALUE_SHAPE`        | `...default_value`, `...default_value.<key>`, `...default_value.value`                                                  | Not an object; an extra key other than `source`/`value`; `literal` without `value`                                                                                                                                                                                              | `none`                                 | §6.5       |
| `INVALID_DEFAULT_SOURCE`             | `...default_value.source`                                                                                               | Absent, `""`, or not `literal` / `refkey`                                                                                                                                                                                                                                       | `none`                                 | §6.5       |
| `INVALID_DEFAULT_REFKEY_SLUG`        | `...default_value.value`                                                                                                | `refkey` and `value` absent, not a string, or blank                                                                                                                                                                                                                             | `none`                                 | §6.5       |
| `INVALID_DEFAULT_REFKEY_TARGET_TYPE` | `...default_value`                                                                                                      | `refkey` and the attribute's `data_type` ≠ `text`                                                                                                                                                                                                                               | `none`                                 | §6.5       |
| `LEGACY_FLAT_PICKLIST_TRIO`          | `$.cfg_data.schema[<n>]`                                                                                                | Any of `picklist_slug`, `picklist_parameters`, `picklist_source_type` present                                                                                                                                                                                                   | `rename_and_retry`                     | §6.5       |
| `INVALID_PICKLIST_BINDING_SHAPE`     | `...picklist`, `...picklist.parameters`                                                                                 | `picklist` not an object; `parameters` present, not `null`, not an array                                                                                                                                                                                                        | `none`                                 | §6.5       |
| `PICKLIST_SLUG_REQUIRED`             | `...picklist.slug`                                                                                                      | Absent or `""`                                                                                                                                                                                                                                                                  | `none`                                 | §6.5       |
| `INVALID_PICKLIST_PARAMETER_SHAPE`   | `...picklist.parameters[<m>]`, `.name`, `.value`                                                                        | Entry not an object; `name` absent or not matching `^p_[a-z][a-z0-9_]*$`; `value` absent, not a string or empty                                                                                                                                                                 | `none` (`rename_and_retry` for `name`) | §6.5       |
| `INVALID_RECORD_LABEL_ATTRIBUTE`     | `...picklist.record_label_attribute`                                                                                    | Present and not a non-blank string matching `^[A-Za-z_][A-Za-z0-9_]*(\.[A-Za-z_][A-Za-z0-9_]*)*$`                                                                                                                                                                               | `rename_and_retry`                     | §6.5       |
| `UNIQUE_ATTR_NAMES`                  | `$.cfg_data.schema`                                                                                                     | Two or more entries share a non-empty `name`, case-insensitively                                                                                                                                                                                                                | `rename_and_retry`                     | §6.6       |
| `DUPLICATE_ALIAS`                    | `$.cfg_data.schema`                                                                                                     | Two or more `fk_config` entries share an alias (alias, else target_domain)                                                                                                                                                                                                      | `rename_and_retry`                     | §6.6       |
| `CHANGE_TRACKING_INVALID_COLUMN`     | `...change_tracking.include_cols[<m>]`, `.exclude_cols[<m>]`                                                            | Element is `null` or not a `schema[]` `column_name`                                                                                                                                                                                                                             | `none`                                 | §6.7       |
| `INDEX_INVALID_COLUMN`               | `$.cfg_data.indexes[<n>].columns[<m>]`                                                                                  | Element is neither a `schema[]` `column_name` nor a meta index column (§7)                                                                                                                                                                                                      | `none`                                 | §6.8       |
| `INDEX_DUPLICATE_DEFINITION`         | `$.cfg_data.indexes[<n>]`                                                                                               | Same ordered `columns` as an earlier index                                                                                                                                                                                                                                      | `none`                                 | §6.8       |
| `TYPE_CHANGE_NOT_ALLOWED`            | `...schema[<n>].data_type`                                                                                              | Storage exists and the column's actual type differs from `data_type` after normalization (always fires for `serial` once storage exists)                                                                                                                                        | `drop_and_recreate`                    | §6.9       |
| `IMMUTABLE_DATA_TYPE`                | `...schema[<n>].data_type`                                                                                              | Published `data_type` for that `column_name` differs                                                                                                                                                                                                                            | `drop_and_recreate`                    | §6.9       |
| `IMMUTABLE_TRACK_VERSIONS_TRUE`      | `$.cfg_data.track_versions`                                                                                             | Published `true`, now `false`                                                                                                                                                                                                                                                   | `drop_and_recreate`                    | §6.9       |
| `IMMUTABLE_XREF_ENABLED_TRUE`        | `$.cfg_data.xref_enabled`                                                                                               | Published `true`, now `false`                                                                                                                                                                                                                                                   | `drop_and_recreate`                    | §6.9       |
| `IMMUTABLE_MULTI_TENANT_FALSE`       | `$.cfg_data.multi_tenant`                                                                                               | Published `false`, now `true`                                                                                                                                                                                                                                                   | `drop_and_recreate`                    | §6.9       |
| `IMMUTABLE_DEDUPE_TRUE`              | `$.cfg_data.dedupe_enabled`                                                                                             | Published `true`, now `false`                                                                                                                                                                                                                                                   | `drop_and_recreate`                    | §6.9       |
| `IMMUTABLE_SLUG` (validator form)    | `$.slug`                                                                                                                | Defined but cannot occur through these endpoints — a published-slug rename is rejected earlier with the `message`-suffix form in §11.1                                                                                                                                          | `drop_and_recreate`                    | §6.9       |
| `DUPLICATE_ATTR_UNIQUE`              | `$.cfg_data.indexes[<n>]`                                                                                               | Warning only (§6.8) — never returned; the save succeeds                                                                                                                                                                                                                         | `none`                                 | §6.8       |

## 12. Coverage checklist

| Inventory # | Method & path                                     | Section |
| ----------- | ------------------------------------------------- | ------- |
| 1           | `GET /api/v3/config-items?itemType=domain`        | §8.1    |
| 2           | `GET /api/v3/config-items/domain/by-slug/{slug}`  | §8.2    |
| 3           | `GET /api/v3/config-items/domain/{id}`            | §8.3    |
| 7           | `GET /api/v3/config-items/domain`                 | §8.4    |
| 4           | `POST /api/v3/config-items/domain`                | §8.5    |
| 5           | `PATCH /api/v3/config-items/domain/{id}`          | §8.6    |
| 6           | `DELETE /api/v3/config-items/domain/{id}`         | §8.7    |
| 12          | `POST /api/v3/domain-publish/{slug}`              | §9.1    |
| 9           | `GET /api/v3/domain-publications/published`       | §9.2    |
| 10          | `GET /api/v3/domain-publications/{id}`            | §9.3    |
| 11          | `GET /api/v3/domain-publications/domain/{domain}` | §9.4    |
| 14          | `GET /api/v3/domain-info`                         | §10.1   |

12 of 12 selected endpoints are documented.

## 13. Excluded from this document

Discovered in the inventory and not selected:

| #   | Method & path                                | Description                                                |
| --- | -------------------------------------------- | ---------------------------------------------------------- |
| 8   | `GET /api/v3/domain-publications`            | List all domain publications                               |
| 13  | `GET /api/v3/domain-info/{domain}/{version}` | Published domain info plus system info for one version     |
| 15  | `POST /api/v3/data-model/catalog`            | Domain catalog for selected domains or areas               |
| 16  | `GET /api/v3/data-model/relationships`       | All authorized domain relationships                        |
| 17  | `DELETE /api/v3/domain/{domain}`             | Permanently delete a published domain and its data objects |
| 18  | `GET /api/v3/table-col-datatype?table=…`     | Column data types of a domain table                        |
| 19  | `POST /api/v3/domain/{domain}`               | Write a record (domain operation)                          |
| 20  | `GET /api/v3/domain/{domain}/{id}`           | Read one record by id                                      |
| 21  | `POST /api/v3/core-data-write`               | Single or bundled record write                             |
| 22  | `POST /api/v3/core-data-read`                | Read records, paginated                                    |
| 23  | `POST /api/v3/core-data-egress`              | Egress read of records                                     |
| 24  | `GET /api/v3/xref/{domain}/{coreRecordId}`   | Cross-references of one record                             |
| 25  | `POST /api/v3/domain-ai-assist/chat`         | AI chat for authoring domain JSON                          |
| 26  | `GET /api/v3/views/{viewName}`               | Older way to read a view by name                           |
| 27  | `GET /api/v3/views/{viewName}/{id}`          | Older way to read one view record                          |

Endpoints that only take a domain as an input (record attachments, per-record process-engine operations, app-builder record routes, ingest loads, transactions and event actions) belong to those resources' references.

## 14. Open questions / notes

**Revision.** `doc_revision` 2026-09-28. This document was generated fresh from the backend and database code in this run; no content was carried over from any earlier revision.

**Verification.** An independent re-derivation against the code was performed on the draft (request schemas bound to each route, the declared domain schema and rule list, the running save step, validator, publish process and its steps, publication and registry views, the error mapper and handler), covering field and rule content, declared-vs-running reconciliation in both directions, error status and wire location, example validity, effective contracts, nested sub-fields, request-layer strictness, enforcement stages, parameter bounds, layout and leaks. It confirmed the function, view and schema versions used are current, every validator code and predicate in §11.3, the combined slug rule and all constant sets, the save/rename/injection behaviour, the save and publish error mapping (including the 200 failure forms), the publication and registry reads, and the leak audit. Defects it reported were fixed in this revision: the type-mismatch 404 text, the validator-exception form of the save error, the `serial` / `TYPE_CHANGE_NOT_ALLOWED` interaction, the defaults of absent capability flags, the publication `enabled` value and failed-run phase, the enqueue-failure `meta`, the publish echo shape, request-layer messages and `full_path: null`, the non-integer issue code, `system_slug` coercion, the §8.3 key order, and additional declared-vs-running conflicts (item 5 below). The fixes were then re-checked against the same code.

**Declared rules vs the running validator** (the platform states the domain contract twice; they were merged, and the code wins where they differ):

1. _Enforced but not declared._ `TRIGGERS_VERSION_CHANGE_REQUIRED`, `INVALID_DEFAULT_VALUE_SHAPE`, `INVALID_DEFAULT_SOURCE`, `INVALID_DEFAULT_REFKEY_SLUG`, `INVALID_DEFAULT_REFKEY_TARGET_TYPE` and `INVALID_READ_VIEW_SLUG` have no declared rule (and `default_value` / `read_view_slug` are not in the declared schema at all); the structural codes `REQUIRED`, `TYPE`, `VALUE`, `INVALID` are also validator-only. Documented from the code; the declared rule list could be back-filled.
2. _Declared but unreachable as declared._ `IMMUTABLE_SLUG` is declared as a validator rule; in practice a published-slug rename is rejected earlier by the save step with a 500 whose `message` ends `Code: IMMUTABLE_SLUG` (not a `details[]` entry). Documented as it reaches the caller.
3. _Different strictness._
   - Declared attribute-name pattern `^(?!_)(?!p_)[A-Za-z0-9_]+$`; the validator uses `^(?!_)[A-Za-z0-9_]+$` plus a separate, case-insensitive `RESERVED_PREFIX_P` check (so `P_x` is also rejected). A leading `_` raises both `VALUE` and `RESERVED_PREFIX_UNDERSCORE`.
   - Declared slug pattern `^[a-z0-9](?:[a-z0-9_]*[a-z0-9])?$`; storage additionally requires a leading letter, so the combined rule is `^[a-z](?:[a-z0-9_]*[a-z0-9])?$`.
   - Declared `SystemAccessEntry.system_slug` pattern and 1–50 length; the validator checks only non-empty.
   - Declared `Attribute.conflict_resolution.code` as required; the validator accepts an object without `code`.
   - Declared `archive_policy` requires both `max_record_versions` and `retention_days`; the validator checks each only when present.
   - Declared `Index.unique` required; not required at save.
4. _Declared structure not enforced at save_ (documented as "expected; not enforced"): `additionalProperties: false` on `cfg_data` and nested objects; `domain_table_name` pattern/length (only equality with the slug is checked, which implies the slug rules); `contact` shape; `fk_config` sub-fields and the `resolution_policy` enum; `json_doc` structure and its field-type enum; `masking.mask_format`; `nullable` / `unique` / `indexed` / `description` / `notes` types.
5. _Declared default or wording that the code contradicts._ `triggers_version_change` is declared optional with default `false`, but the validator **requires** it on every attribute (`TRIGGERS_VERSION_CHANGE_REQUIRED`). `track_versions` is declared with default `true`, but save and the registry treat an absent value as `false`, so `xref_enabled: true` without `track_versions` fails `XREF_REQUIRES_TRACK`. `default_conflict_resolution` is declared as keeping its prior value when omitted on edit, but update replaces `cfg_data` wholesale, so omitting it fails `REQUIRED`. The declared attribute `sys_` and `p_` prefix rules name the lowercase prefixes literally; the validator checks both case-insensitively on `name` and `column_name`.
6. _Declared warning._ `DUPLICATE_ATTR_UNIQUE` is declared as a warning and implemented as one; warnings are discarded by the save step and never reach the caller.

**Request layer vs platform** (the combined, stricter rule is what §6 / §8 document):

- `slug`: request layer 1–255 chars; platform ≤ 50 + pattern + reserved prefixes + global uniqueness + equality with `domain_table_name`.
- `name`: optional at the request layer (not nullable); required by the validator, satisfied on create by defaulting to the slug.
- `enabled`: nullable at the request layer; `null` on update violates storage (500).
- `full_path`: empty on update violates storage (500).
- `version`: validated at the request layer and then ignored.
- Update reuses the create body shape, so `slug`, `item_type`, `cfg_data` are required-to-send although the platform patch-merges omitted fields.
- `cfg_data.access_control` is accepted by the request layer and overwritten by the platform.

**Behaviours worth confirming with the platform owners** (documented as the code behaves today):

1. Soft-delete (§8.7) only disables the item; no domain is ever marked deleted through this API, and a disabled domain can still be published. Its published storage is untouched.
2. `PATCH` with an unknown uuid creates a new domain with that id (§8.6).
3. All platform validation and state rejections return 500; codes are not returned except as `details[].code` on save validation and the `IMMUTABLE_SLUG` message suffix (§11).
4. Authorization failures return a non-envelope `{ "error": ... }` body (§3).
5. Publish (§9.1): `data.pub_id`, `data.statuses.*` are always `""` and `data.has_warnings` is always `false`; the real values are in `meta` / `data.response`. A failed create step can be reported as 200 with a `data.response` that is the stored domain item wrapped in internal input keys, with no `result` key, and an unexpected error after the create step starts is reported as 200 with `result` `success`/`warning`. The `meta` key `archive result` contains a space.
6. `is_published` (§8.4) and `publish_status` (§8.3) treat a finished run as published even when a step failed; `/published` (§9.2) excludes runs whose create step ended in `warning`.
7. The publication record shows an absent `archive` / `xref_enabled` as `true`, while save checks and the published-domain registry treat it as `false` (§6.2 warning).
8. Duplicate `column_name` values across `schema[]` are not detected at save.
9. Validation warnings (`DUPLICATE_ATTR_UNIQUE`) are silently dropped.
10. Publish has no guard against overlapping runs for the same slug.
11. Some platform failure texts that reach the caller name internal storage or processes (publish lookup failure, enqueue diagnostics); this document describes them rather than quoting them.

**⚠️ Not determinable from source:** the publish-time effect of `nullable`, `indexed`, `fk_config` and `masking`; the inner structure of publication step reports and of `colTypes` / `fkResolverCfg`; which publication §8.3 reports when several share a version; the order of same-version rows in §9.4; whether the §8.1 `parentItemId` filter can match; the offset used in save-response timestamps.
