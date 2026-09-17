---
name: hello-world
description: Confirm the Pivotly plugin is installed and report its status — version, last updated date, loaded skills, and server connection.
category: core
---

# Hello World

First contact with the Pivotly plugin. Says hello, then reports what is actually installed
and what is actually answering.

## When to use

- Someone asks whether the Pivotly plugin is installed, working, or connected.
- Someone asks which version this is, when it was last updated, or which skills it has.
- Right after installing the plugin, to verify the install took.
- A tool call failed and it is not yet clear whether the plugin, the server, or the
  contract between them is at fault.

## Instructions

1. Greet once, name the plugin, then report status. A bare "Hello!" answers nothing that
   the person could not already see.
2. Read every field from its source at the moment of the report. A version quoted from
   earlier in the conversation is a guess about the present.

   | Field | Where it comes from |
   | --- | --- |
   | Name and version | The manifest installed beside this skill: `.claude-plugin/plugin.json` in a Claude Code or Cowork plugin, `gemini-extension.json` in a Gemini extension. A client that received copied-in files has no manifest — report "unknown". |
   | Last updated | The plugin manager's or marketplace's own "last updated" or version field, when the client shows one. Otherwise "unknown". |
   | Skills | The `SKILL.md` files (or command files) installed alongside this one. List them by name. |
   | Server build and protocol | The `pivotly_server_info` tool: `server_version`, `protocol_version`, and the tool inventory it lists. |
   | Connection | The `pivotly_ping` tool. `pong: true` with your message echoed back means the server answered, using this plugin's credentials. |

3. Separate what is on disk from what is live. The first three rows are true of an install
   that has never reached a server; the last two are claims about a running one.
4. Report the connection only if it was attempted this session. "Not attempted" is a real
   answer and a useful one — it tells the reader which half is still unverified.
5. Say what is missing rather than omitting the row. A field with no source found is
   "unknown", not absent.
6. Keep it to one block. Status is read at a glance, and prose hides the one line that
   changed.

## Pitfalls

- **An installed plugin is markdown and JSON, nothing else.** There is no `package.json`,
  no source code, and no library to import. Do not go looking for them; a search that
  comes back empty is not a broken install.
- **A file timestamp is not an update date.** `mtime` is rewritten by an install, a sync,
  or a build, so it reports the day the machine touched the file rather than the day the
  plugin changed. Without a real "last updated" field the honest answer is "unknown" — an
  mtime dressed up as a release date is worse than no date.
- **`0.0.0` is a placeholder, not a release.** Reporting it as a version implies a release
  that was never cut; say it is unreleased.
- **The plugin's version is not the server's.** They deploy independently, so a plugin can
  be newer or older than the server it points at. `pivotly_server_info` is what names the
  server side; a status report that lists one version invites the reader to assume both
  sides are on it.
- **Installed is not connected.** Files on disk prove the install was written and nothing
  else. Only `pivotly_ping` proves a server answered.
- **Never print the credential.** The server's host name belongs in a status report; the
  `Authorization` header value in the plugin's MCP config does not — whether it is a
  literal token or an unexpanded expression — because a status block is pasted into issues.
