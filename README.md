# Pivotly client setup

@pivotly/plugin 0.0.0 · sha256:b50e3f2e5b60

Pinned to http://localhost:8080/mcp — nothing to set.

This build sends no credential — it targets a server with auth disabled.

## Install

**Claude Code / Cowork plugin**

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

**Cowork (Claude Desktop)**

Under **Customize → Plugins**, choose **Add marketplace** and enter this repository's GitHub URL, then install `pivotly` from it. Plugins are not used in Chat.

## Unverified config paths

A best reading of each client, not confirmed against a running install. If one is
wrong the client ignores the file: the skills work, the tools never appear, and
nothing reports an error.

- OpenAI Codex — codex/.codex/config.toml
- GitHub Copilot plugin — copilot/mcp.json
- Windsurf — windsurf/.windsurf/mcp_config.json

Skills: hello-world
