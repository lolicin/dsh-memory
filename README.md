# dsh-memory

Durable cross-session memory for [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness).

This is a fork of `dsh-memory` that widens the peer range for DSH 0.2.0-rc.2 and fixes CJK search. The harness ships no memory plugin. Its `extension-cookbook` names the mechanism — a prompt section plus tools — but nothing implements it, so every session starts blank. This package fills that gap with one local SQLite file: **no embedding service, no API key, no sidecar process.**

## Install

```bash
dsh plugin --profile web add dsh-memory
```

The shipped bundle row stores memories at `$DSH_HOME/memory/memory.db`, shared by every profile on the machine.

## What it gives the model

| Tool | Purpose |
|---|---|
| `memory_write` | Store one self-contained durable fact, optionally tagged and pinned |
| `memory_search` | Keyword search over memory text and tags |
| `memory_forget` | Delete a memory that is now wrong or obsolete |

Plus a `memory:recall` prompt section that renders **pinned memories first, then the most recently updated**, under a character budget. Recall therefore does not depend on the model remembering to search — what it stored is already in front of it, and search is for anything older than the budget allows.

The `memory_write` description steers the model away from the common failure modes: transient task state (that is what the todo list is for), secrets, and facts the repository already records.

## Configuration

```yaml
- id: memory
  name: dsh-memory
  config:
    path: !!js dshHomePath('memory/memory.db')
    promptRecentCount: 10
    promptMaxChars: 2000
    maxTextChars: 2000
    searchLimitDefault: 10
    searchLimitMax: 50
    promptOrder: 50
```

| Field | Default | Meaning |
|---|---|---|
| `path` | — (required) | SQLite file, or `:memory:` for an ephemeral store |
| `promptRecentCount` | `10` | Unpinned recent memories offered to the prompt section |
| `promptMaxChars` | `2000` | Budget for the rendered section; overflow is reported as a count, and pinned memories are emitted first so they survive a tight budget |
| `maxTextChars` | `2000` | Maximum characters accepted for one memory |
| `searchLimitDefault` | `10` | `memory_search` limit when the model omits it |
| `searchLimitMax` | `50` | Hard cap, whatever the model asks for |
| `promptOrder` | `50` | Section order; `-100` is the harness identity, `0` the persona |

`path` has **no code-side default on purpose**: a default would scatter durable user facts into whatever directory the harness happened to start in. The deployment value lives in the patch row.

## Storage

One SQLite file: a `memories` table plus two external-content FTS5 indexes kept in sync by triggers. Parent directories are created on open, and the store survives process restarts.

### CJK search

FTS5's `unicode61` tokenizer treats a run of CJK as a single token, so a Chinese keyword can only match a memory storing the exact same character run — `机骸` misses `机骸第九行星`, and a model searching for a word that sits mid-sentence finds nothing. Search therefore tries three strategies in order:

1. the `unicode61` index, which answers latin and whole-run queries with its original ranking;
2. a `trigram` index for CJK tokens of three or more characters, which matches a fragment anywhere inside a run;
3. a `LIKE` sweep for the one- and two-character cases trigram cannot express, requiring every token to appear in the text or tags.

The last two only run when the first found nothing, so the common latin path still costs one query. The sweep is a table scan bounded by the user's own memory count, which stays small. A store created before this version gains the trigram index on first open: the index is rebuilt whenever its row count disagrees with `memories`, so no data is lost and no manual migration is needed.

Search splits the query on non-alphanumeric characters and **quotes every token**, so FTS5 operators a model happens to type (`OR`, `*`, `-`) are matched literally instead of changing the query's meaning or raising a syntax error mid-tool-call. Surviving tokens combine with FTS5's implicit AND: every token must appear, and a query whose tokens include a word you did not store legitimately matches nothing. Note that a character a model may intend as punctuation — `a"b` is a single token — becomes a separator, so such a query is read as `a AND b`.

`node:sqlite` is still flagged experimental in Node 22/24, so running the harness prints one `ExperimentalWarning`. The harness's own `dsh-session-query-sqlite` uses the same module.

## Failure behavior

Load-time misconfiguration fails loud: an empty `path`, a non-positive bound, or a `searchLimitDefault` above `searchLimitMax` throws at plugin load.

At call time, a blank fact or one over `maxTextChars` is a tool error the model can correct. A `memory_forget` for an id that does not exist is a **successful** result reporting `forgotten: false` — the model asked for a state that already holds, which is not an infrastructure failure.

## Extension points

`ctx.tools.register()` for the three tools and `ctx.systemPrompt.section()` for recall. Every registration is a Cordis effect, so unloading the plugin removes the tools and the section together and closes the database.

## Development

```bash
pnpm install --ignore-workspace
pnpm run typecheck
pnpm test
pnpm run build
```

Tests cover the store directly (FTS retrieval, the literal-token query contract, prompt ordering, durability across reopens) and the plugin against the **real** tool registry and prompt service (registration, disposal, the write→recall round trip, bounds, fail-loud config).

## Fork changes

- **Peer range** widened to `^0.1.0-rc.6 || ^0.1.7-rc.2` so the bundle is not skipped by the DSH 0.2.0-rc.2 compatibility gate. Verified against the installed `dsh-tools@0.1.7-rc.2`: `apply`, the three tool shapes, `presentCall` and `output.render` are all unchanged, so no runtime code needed touching.
- **CJK search** fixed as described above; schema version bumped to 2 and the trigram index backfills itself on open.

## License

MIT

## Prior art

The idea comes from [pi-mentis](https://github.com/guchengod/pi-mentis) (MIT) in the Pi ecosystem. This is an independent implementation against Harness extension points and shares no code with it. It deliberately drops pi-mentis's sidecar process, Zvec vector store, and required SiliconFlow embedding key in favour of one local FTS5 file — smaller, keyless, and offline, at the cost of lexical rather than semantic retrieval.
