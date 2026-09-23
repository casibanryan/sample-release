# Pivotly

This extension adds the Pivotly MCP server and the commands below.

## Commands

- `/pivotly:domain` — Author, publish, inspect, and retire Pivotly domains (governed, versioned data entities) through the config-item_domain MCP tool — create or update a domain's cfg_data schema, publish it to build its live data structures, check publication runs and metadata, and soft-delete the config item or permanently delete the published domain, confirming before publish and every other mutation. Use whenever someone mentions a Pivotly domain, even casually or indirectly: 'create a products domain', 'make a new domain with these fields', 'add a column to the customers domain', 'change conflict resolution to first write wins', 'turn on version tracking for the domain', 'publish the orders domain', 'push my domain changes live', 'did the publish go through', 'why did my publish fail', 'validation failed when saving my domain', 'show me the domain config', 'what domains do we have', 'which domains are published', 'find domains matching inventory', 'what columns does the vendors domain have', 'is anyone publishing right now', 'delete the test domain', 'retire this domain', 'remove the published domain and its data'.
- `/pivotly:hello-world` — Confirm the Pivotly plugin is installed and report its status — version, last updated date, loaded skills, and server connection.

Each command carries its own procedure and pitfalls; invoke one rather than guessing.
