---
name: hello-world
description: Confirm the Pivotly plugin is installed and connected — greet, then report the plugin's details, the MCP server's connection status, and the tools it exposes.
category: core
---

# Hello World

First contact with the Pivotly plugin. Says hello, then reports what this host has actually
loaded: the plugin itself, the Pivotly MCP server, and the tools that server exposes.

## When to use

- Someone asks whether the Pivotly plugin is installed, working, or connected.
- Someone asks which Pivotly tools are available, or why a Pivotly tool cannot be called.
- Right after installing the plugin or restarting the host, to verify the install took.
- A tool call failed and it is not yet clear whether the plugin or the server is at fault.

## Instructions

1. Greet once, name the plugin, then report status. A bare "Hello!" answers nothing the
   person could not already see.
2. Answer from the RUNNING SESSION, never from the filesystem. Every field below is
   something the host already knows because it loaded the plugin; read it there.

   | Field | Where it comes from |
   | --- | --- |
   | Name, version, description | the installed plugin's metadata, as this host reports it |
   | Skills | the skills this host lists under the plugin |
   | MCP server | the `pivotly-mcp-server` entry the host loaded from its MCP config |
   | Connection | whether the host reports that server as connected, or names a failure |
   | Tools | the tools the host lists for that server, by their exact names |

3. List the tools the host actually shows, not the ones this file names. The server's
   inventory changes on its own schedule; this document is a description of it from the
   day it was written. At time of writing the surface is `pivotly_ping` and
   `pivotly_server_info`, exposed by the host under its own MCP prefix.
4. State connected or not connected in as many words, and when it is not, report the
   host's own reason verbatim (a rejected token, an unreachable URL, a server that never
   started). A paraphrased error sends the reader to the wrong fix.
5. No tools listed means one of two things, and they need different fixes: the server is
   connected and advertises none, or it never connected. Say which.
6. Call `pivotly_ping` only when the server is already listed as connected and the person
   wants liveness confirmed. It answers "does it respond", not "is it configured", and it
   proves nothing about a server the host could not reach.
7. Keep it to one block. Status is read at a glance, and prose hides the one line that
   changed.

## Pitfalls

- **Installed is not connected.** A plugin can load, mount its skills, and expose no tools
  at all because its server rejected the connection. Report the two separately; a single
  green checkmark over both is the claim that gets people stuck.
- **Do not go looking on disk.** Opening `.mcp.json`, `plugin.json`, `package.json`, or a
  git log answers a different question — what a checkout on this machine contains, which
  may be a different build from the one the host has loaded, or no relation to it at all.
  If a field is not visible in the session, it is "unknown".
- **The plugin's version is not the server's.** They deploy independently, so a host can
  run a build compiled against a newer contract than the server it points at. Label which
  version belongs to which side; one unlabeled number invites the reader to assume both.
- **`0.0.0` is a placeholder, not a release.** The package is `private` and pinned there.
  Reporting it as a version implies a release that was never cut; say it is unreleased.
- **Never print the token.** The server's name and URL belong in a status report; the
  `Authorization` header and `PIVOTLY_MCP_TOKEN` do not, and a status block is pasted into
  issues. When a token is the problem, say the token was rejected — never quote it.
