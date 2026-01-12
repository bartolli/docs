# Codanna Documentation

Documentation for Codanna, built with Mintlify.

## Structure

```
docs/
├── index.mdx            # Overview
├── installation.mdx     # Installation + Quick Start
├── concepts/            # Core concepts
│   ├── symbols.mdx      # Symbols, search, performance
│   ├── embeddings.mdx   # Semantic search models
│   ├── documents.mdx    # Document search (RAG)
│   ├── plugins.mdx      # Plugin system
│   └── profiles.mdx     # Profile system
├── cli/
│   └── reference.mdx    # CLI reference
├── mcp/
│   └── tools.mdx        # MCP tools reference
├── integration/
│   ├── claude-code.mdx  # Claude Code + HTTP
│   ├── codex-cli.mdx    # Codex CLI
│   ├── gemini-cli.mdx   # Gemini CLI
│   ├── piping.mdx       # Unix piping
│   └── agent-workflows.mdx  # Agent guidance
└── contribute.mdx       # Contributing guide
```

## Development

Install Mintlify CLI:

```bash
npm i -g mint
```

Run local preview:

```bash
cd documentation/docs
mint dev
```

View at `http://localhost:3000`.

## Publishing

Changes deploy automatically after pushing to main branch.
