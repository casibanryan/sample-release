# Pivotly client setup

@pivotly/plugin 0.0.0 · sha256:af2c18056f21

Pinned to https://app-sbx-mcp-authtest-cus-001.azurewebsites.net/mcp — nothing to set.

Sign-in happens in your client: the first connection opens a browser window (in
Claude, via the `npx mcp-remote` bridge, which needs Node). There is no token to set.

## Install

**Claude Code plugin**

```bash
claude plugin marketplace add . && claude plugin install pivotly
```

**Gemini CLI extension**

```bash
gemini extensions install .
```

**OpenAI Codex**

```bash
cp -r codex/. /path/to/project/
```

**Cursor**

```bash
cp -r cursor/. /path/to/project/
```

**GitHub Copilot plugin**

```bash
copilot plugin marketplace add . && copilot plugin install pivotly@pivotly
```

**Windsurf**

```bash
cp -r windsurf/. /path/to/project/
```

## Unverified config paths

A best reading of each client, not confirmed against a running install. If one is
wrong the client ignores the file: the skills work, the tools never appear, and
nothing reports an error.

- OpenAI Codex — codex/.codex/config.toml
- GitHub Copilot plugin — copilot/mcp.json
- Windsurf — windsurf/.windsurf/mcp_config.json

Skills: hello-world, manage-data-view, manage-domain
