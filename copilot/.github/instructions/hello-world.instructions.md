---
description: Confirm the Pivotly plugin is installed and report its status — version, last updated date, loaded skills, and server connection.
applyTo: "**"
---

# Hello World

First contact with the Pivotly plugin. Says hello, then reports what is actually running.

## When to use

- Someone asks whether the Pivotly plugin is installed, working, or connected.
- Someone asks which version this is, when it was last updated, or which skills it has.
- Right after generating or installing a client package, to verify the install took.
- A tool call failed and it is not yet clear whether the plugin, the server, or the
  contract between them is at fault.

## Instructions

1. Greet once, name the plugin, then report status. A bare "Hello!" answers nothing that
   the person could not already see.
2. Read every field from its source at the moment of the report. A version quoted from
   earlier in the conversation is a guess about the present.

   | Field | Where it comes from |
   | --- | --- |
   | Name and version | `version` in the plugin package's `package.json` |
   | Last updated | `git log -1 --date=short --format='%cd %h %s'` over the plugin directory |
   | Protocol version | `PivotlyPlugin.protocolVersion` — always the contract's copy |
   | Skills | `loadCoreSkills()` and `loadCustomSkills()`, or the directories under `skills/library/<category>/` |
   | Server | `serverUrl` and `authMode` from the resolved `PluginConfig` |
   | Connection | `await plugin.ping()`, then `await plugin.assertContractMatch()` |

3. Separate what is on disk from what is live. The first four fields are true of an
   install that has never reached a server; the last two are claims about a running one.
4. Report the connection only if it was attempted this session. "Not attempted" is a real
   answer and a useful one — it tells the reader which half is still unverified.
5. Say what is missing rather than omitting the row. A field with no source found is
   "unknown", not absent.
6. Keep it to one block. Status is read at a glance, and prose hides the one line that
   changed.
