# Pivotly client setup

@pivotly/plugin 0.0.0 · sha256:5a65fb665660

Pinned to http://localhost:8080/mcp — nothing to set.

This build sends no credential — it targets a server with auth disabled.

## Install

**Claude Code plugin**

```bash
claude plugin marketplace add . && claude plugin install pivotly
```

**Gemini CLI extension**

```bash
gemini extensions install ./gemini
```

**OpenAI Codex**

```bash
cp -r codex/. /path/to/project/
```

**Cursor**

```bash
cp -r cursor/. /path/to/project/
```

**GitHub Copilot**

```bash
cp -r copilot/. /path/to/project/
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
- GitHub Copilot — copilot/.vscode/mcp.json
- Windsurf — windsurf/.windsurf/mcp_config.json

Skills: greeting, math-add, math-divide, math-multiply, math-subtract
