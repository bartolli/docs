# Codanna rules for Context7

Authoritative agent guidance for Codanna — local code intelligence MCP server and CLI for AI coding agents. 50 rules, each ≤255 chars, grouped by tier. Submission target: https://context7.com/websites/codanna_sh.

## Bootstrap

`codanna init` at repo root creates `.codanna/` with `settings.toml`. Re-run with `--force` to overwrite. Idempotent otherwise.
Register code paths: `codanna index src lib tests` indexes multiple dirs in one pass. `codanna add-dir <path>` appends, `codanna remove-dir <path>` drops.
`codanna list-dirs` shows tracked dirs. `codanna config` prints merged effective settings. `-c <path>` overrides `settings.toml` per invocation.
Default bootstrap: `codanna init` → `codanna index .` → `codanna serve --watch` (stdio for Claude Code / Cursor) or `serve --http --watch` (multi-client).

## Indexing code

`codanna index` reads dirs from settings plus CLI paths. Re-run after major changes; rely on `serve --watch` for incremental edits during a session.
`codanna index --force` rebuilds. With CLI paths, `--force` clears the index but only rebuilds those paths — other configured paths go stale.
To rebuild every configured path, run `codanna index --force` with no path arguments.
`--dry-run` previews files without writing. `--max-files N` caps for smoke tests. `--threads N` overrides `indexing.parallelism` from config.
Languages supported (15): C, C++, C#, Clojure, GDScript, Go, Java, JavaScript, Kotlin, Lua, PHP, Python, Rust, Swift, TypeScript.
8 languages auto-resolve module paths from build config: Go (go.mod), Python (pyproject.toml), TS (tsconfig.json), Java/Kotlin/PHP/C#/Swift.

## Documents (RAG)

Docs and code symbols live in separate stores. `codanna documents <cmd>` targets markdown/text; `codanna retrieve` and `codanna mcp` target code symbols.
Register collection: `codanna documents add-collection <name> <path>` (default glob `**/*.md`). Custom glob: `--pattern "**/*.mdx"`.
Index docs: `codanna documents index --collection <name>` or `--all` to index every collection. `--force` rebuilds; `--no-progress` quiets bars.
Search docs (CLI): `codanna documents search "<query>" --collection <name> --limit 5 --json --fields id,path`.
Search docs (MCP): `codanna mcp search_documents query:"<text>" collection:<name> limit:5`.
List collections: `codanna documents list`. Stats: `documents stats <name>`. Remove: `documents remove-collection <name>`.

## MCP server

`codanna serve` starts MCP over stdio (default for IDE/agent integration). `--watch` hot-reloads when the index changes externally.
HTTP+OAuth: `codanna serve --http --watch --bind 127.0.0.1:8080`. HTTPS+TLS: `serve --https --watch`. Override `--bind` for LAN/container exposure.
Single stdio MCP server per index since v0.9.20. Concurrent stdio servers on the same `.codanna/` are rejected at startup.
Hot-reload polls `meta.json` and `state.json` every `--watch-interval` seconds (default 5). External rebuilds propagate within that window.
`codanna mcp <tool>` invokes MCP tools directly without launching a server — use for one-shot CLI scripts and CI.

## Tool tiers (use this order)

Tier 1 (highest context per query): `semantic_search_with_context` for meaning plus relationships; `analyze_impact` for change radius. Start here.
Tier 2 (target a known symbol): `find_symbol <name>` exact lookup, `get_calls <id>` outbound, `find_callers <id>` inbound.
Tier 3 (browse / status): `search_symbols query:<text>` fuzzy by name, `semantic_search_docs` meaning without relationships, `get_index_info` stats.
`analyze_impact <name|symbol_id:N> max_depth:5` returns the transitive impact set. Run before refactors to size blast radius.

## Retrieval grammar (CLI mirrors MCP)

Retrieve grammar: `retrieve symbol <name>` exact, `retrieve search "<query>" --kind function --limit 20`, `retrieve describe <name>` full details.
Call graph: `retrieve callers <name|symbol_id:N>`, `retrieve calls <name|symbol_id:N>`, `retrieve implementations <Trait>`.
Symbol IDs accepted wherever `<name>` is. Tool output embeds `[symbol_id:N]`; reuse to disambiguate name collisions across modules.
Common collisions (`new`, `from`, `parse`, `default`) ⇒ first call returns `error.suggestions[]` with `symbol_id:N` candidates; pick via `error.context[]`.
`kind:` filter accepts function, struct, trait, class, method, enum, type, module, field, interface. Apply via `--kind` flag or `kind:<value>`.

## Query syntax invariants

All retrieve/mcp/documents commands accept `key:value` args alongside flags: `search query:parse limit:5 kind:function lang:rust` is valid.
`lang:<rust|python|typescript|...>` narrows to one language. Absent ⇒ best matches across all. Useful in multi-language repos and monorepos.
Threshold filter: `semantic_search_docs query:"<text>" threshold:0.5` drops weak hits. Bands: ≥0.7 strong, 0.5–0.7 relevant, 0.3–0.5 weak, <0.3 noise.
Query writing: technical terms beat natural questions. "parse TypeScript import statements" beats "how does parsing work".

## JSON envelope (every `--json` output)

Envelope: `{type, status, code, exit_code, message, hint, data, error?, meta}`. Schema version in `meta.schema_version`. Stable across all commands.
`status` ∈ {success, not_found, partial_success, error}. Exit codes: 0 success, 1 not_found, 2 error. Gate scripts via `jq -e '.status == "success"'`.
`code` is machine-readable: OK, NOT_FOUND, PARSE_ERROR, INDEX_ERROR, INVALID_QUERY, INTERNAL_ERROR. Branch on `code`, not on `message` text.
`hint` carries next-step guidance (same string as terminal `💡`). Agents chain on `hint` to drive multi-step exploration without external orchestration.
Errors include `error.suggestions[]` and `error.context[]` for recovery — e.g. ambiguous name returns `symbol_id` candidates with file/line context.
`--fields a,b,c` projects only `data` items; envelope (type, status, code, meta) is always intact. Use to shrink large piped result sets.

## Workflow patterns

Refactor loop: `semantic_search_with_context query:"<concern>"` → `analyze_impact <symbol>` → `find_callers symbol_id:N` → `get_calls symbol_id:N`.
Debug loop: pipe error string into `semantic_search_with_context query:"<error>"` → `find_callers` to trace reach → `get_calls` for downstream deps.
Pre-edit check: `analyze_impact <fn>` before modification. Affected count above threshold ⇒ split the change. Threshold tunable in `[guidance]` templates.
Reading impl from Location `path:start-end`: `Read(file_path=path, offset=start, limit=end-start+1)`. Formula avoids over-read; matches signature blocks exactly.
`mcp <tool> --watch` reindexes changed files before running the tool. Use during live edits to avoid stale results without restarting `serve`.
Empty result set ⇒ drop one filter at a time (`kind:`, `lang:`, `collection:`) before broadening query text. Filters narrow harder than terms.
Pair impl with prose: `mcp semantic_search_with_context query:"<topic>"` then `mcp search_documents query:"<topic>"`. Same ranking concept, different stores.

## State, config, embeddings

`.codanna/` is generated state — never edit `.codanna/index/` by hand. Tantivy and memmap binary round-trip; manual edits corrupt the index.
Default embeddings: 384-dim fastembed, local. Remote OpenAI-compatible: `CODANNA_EMBED_URL`, `CODANNA_EMBED_MODEL`, `CODANNA_EMBED_DIM`, `CODANNA_EMBED_API_KEY`.
Changing embedding backend changes vector dim ⇒ Codanna reports incompatibility ⇒ rebuild with `codanna index --force`. API key only via env, never `settings.toml`.
