# Claude Code setup

`settings.json` in this directory is project-scoped configuration that Claude Code
reads automatically when a session starts in this repository.

## Installed plugins

### caveman — https://github.com/JuliusBrussee/caveman

Ultra-compressed output mode for AI coding agents: same technical accuracy, fewer
output tokens. `settings.json` registers the upstream marketplace and enables the
plugin, so any session in this repo picks it up without a manual install.

The plugin ships its own `SessionStart` / `UserPromptSubmit` hooks via its manifest,
so no hook wiring is needed here.

Useful commands once active: `/caveman`, `/caveman-stats`, `/caveman-review`,
`/caveman-commit`.

To install it for every repo on a machine (user scope) instead of just this one:

```bash
claude plugin marketplace add JuliusBrussee/caveman
claude plugin install caveman@caveman
```

Or use the upstream installer, which also covers non-Claude agents (Cursor, Codex,
Gemini CLI, and others):

```bash
curl -fsSL https://raw.githubusercontent.com/JuliusBrussee/caveman/v2.5.0/install.sh -o install.sh
# review it, then:
bash install.sh
```

Uninstall: `npx -y github:JuliusBrussee/caveman -- --uninstall`

## MCP servers

`../.mcp.json` at the repository root registers project-scoped MCP servers.

### markitdown — https://github.com/microsoft/markitdown

Converts PDF, Office documents (Word/Excel/PowerPoint), HTML, images, audio, EPUB,
ZIP archives and URLs into Markdown suited for LLM consumption. The MCP server
exposes one tool, `convert_to_markdown(uri)`, accepting `http:`, `https:`, `file:`
or `data:` URIs.

It is launched with `uvx markitdown-mcp`, so it needs [uv](https://docs.astral.sh/uv/)
on the machine but no persistent install. To install it permanently instead:

```bash
uv tool install markitdown-mcp     # or: pip install markitdown-mcp
```

The standalone CLI is a separate package:

```bash
uv tool install 'markitdown[all]'  # or: pip install 'markitdown[all]'
markitdown path/to/file.pdf -o out.md
```

`[all]` pulls in every optional converter. Audio transcription additionally needs
`ffmpeg` on `PATH`. The MCP server is intended for local use with trusted agents;
leave it on STDIO or bound to localhost.
