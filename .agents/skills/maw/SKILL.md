---
name: maw
description: Crawl a website, docs site, git repo, or local git checkout into a searchable .mv2 file with maw, then search it offline. Use when the user asks to crawl, scrape, archive, index, or "save" a site or docs for later, wants to search across a whole site's content, asks a question that a specific documentation site would answer, or mentions maw, memvid, or a .mv2 file. Also use when web fetching one page at a time is too slow or too shallow and the whole site needs to be searchable.
license: MIT
compatibility: Requires Node 20 or newer, network access for crawling, and either a global maw install or npx. Optional Playwright for JavaScript-heavy or bot-protected sites.
---

# maw: crawl once, search forever

maw turns a site, repo, or folder into a single `.mv2` file with BM25 (and optionally semantic) search built in. Crawl once, then answer questions from the file instead of re-fetching pages.

## Running it

Pick one invocation prefix up front and use it for every command that follows. `npx` runs a package command once; it does not put `maw` on `PATH`, so a bare `maw` after an npx fallback will fail.

```bash
if command -v maw >/dev/null 2>&1; then MAW="maw"; else MAW="npx @memvid/maw"; fi
$MAW --version
```

Every example below writes `maw`; substitute `$MAW` (or `npx @memvid/maw`) when there is no global install. Inside the maw source repository, `npx tsx bin/maw.ts` also works without a build.

Requires Node 20 or newer.

## Workflow

1. **Preview before a big crawl.** Shows the sitemap and a page-count estimate so you can pick sensible limits.
   ```bash
   maw preview https://docs.example.com --json
   ```
2. **Crawl.** Depth and max-pages are auto-detected (single page: depth 0; domain root: depth 2, 150 pages). Raise them only when the preview shows more pages worth having.
   ```bash
   maw https://docs.example.com -o docs.mv2
   maw https://docs.example.com -o docs.mv2 --depth 4 --max-pages 1000
   maw https://github.com/org/repo -o repo.mv2      # any git repo
   maw . -o code.mv2                                  # local git repo root only
   maw https://a.dev https://b.dev -o both.mv2        # several sources, one file
   ```
   Passing an existing `.mv2` path as a bare argument (`maw https://x.dev docs.mv2`) appends to it.

   A local path is only read from disk when that directory itself contains a `.git` entry. A plain folder or a subdirectory such as `./src` is treated as a hostname and crawled over HTTPS, so run maw from the repository root, or pass a path to one. There is no folder-ingestion mode for non-git directories.
3. **Search.** Always pass `--json` when the output is for you rather than the user; it is easier to read reliably than the styled table.
   ```bash
   maw find docs.mv2 "useEffect cleanup" --json -k 10
   maw list docs.mv2 --json -l 50
   ```
4. **Answer from the hits yourself.** `maw find --json` returns chunk text and source URLs. Read them and write the answer. Do not reach for `maw ask` when you are the agent: it sends the question to an OpenAI model and needs `OPENAI_API_KEY`, which is redundant work and an extra dependency. Only use `maw ask` when the user explicitly wants it.
5. **Export when the user wants the content outside maw.**
   ```bash
   maw export docs.mv2 -f markdown --out docs.md     # also: json, csv
   ```

## Flags the crawl command accepts

| Flag | Effect | Default |
|---|---|---|
| `-o, --output <file>` | Output file | `maw.mv2` |
| `-d, --depth <n>` | Link depth | auto |
| `-m, --max-pages <n>` | Stop after n pages | 150 |
| `-c, --concurrency <n>` | Parallel requests | 10 |
| `-r, --rate-limit <n>` | Requests per second | 10 |
| `-t, --timeout <ms>` | Per-request timeout | 10000 |
| `--include <regex>` | Only crawl matching URLs | |
| `--exclude <regex>` | Skip matching URLs | |
| `--label <label>` | Label stored on ingested docs | `web` |
| `--no-sitemap` | Skip sitemap.xml discovery | |
| `--no-robots` | Ignore robots.txt | |
| `--browser` | Force Playwright (JS-heavy sites) | |
| `--stealth` | Force rebrowser (anti-bot sites) | |
| `--embed [model]` | Semantic embeddings: `bge-small`, `openai`, `nvidia` | off; `bge-small` if no model given |
| `--memory <id>` | Bind to a memvid cloud memory | |
| `-q` / `-v` | Quiet or verbose output | |

Engine fallback is automatic (fetch, then Playwright, then rebrowser). Only force `--browser` or `--stealth` when a plain crawl visibly returned empty or blocked pages. Playwright and rebrowser are optional dependencies; if they are missing, the fallback will not fire.

## Judgment calls

- **Scope the crawl.** Use `--include` to stay inside a docs subtree (for example `--include "/docs/"`) rather than crawling a whole marketing site. Use `--exclude` to drop changelogs, blog archives, or localized duplicates.
- **Be polite by default.** Keep `--rate-limit` at 10 or lower on small sites, and use `--rate-limit 2` on anything that looks like a personal or low-traffic server.
- **Do not pass `--no-robots` on your own.** Respecting robots.txt is the default and the user has to ask to override it.
- **Free tier limit is 50 MB per file** (roughly 500 to 2000 pages). If a crawl reports it stopped at the limit, tell the user rather than silently splitting into more files.
- **Embeddings are optional.** BM25 is enough for docs lookups. Suggest `--embed` only when keyword search is clearly failing on meaning-based questions, and note that `openai` costs money and needs `OPENAI_API_KEY`.
- **`OPENAI_API_KEY` changes `find` behavior.** When it is set, `maw find` embeds the query with OpenAI and runs hybrid search, which costs money and adds latency. It falls back to lexical search on a dimension mismatch. Unset the key for the command (`env -u OPENAI_API_KEY maw find ...`) when you only want BM25.
- **`.mv2` files are gitignored** in this repo and are usually large. Keep them out of commits unless the user says otherwise.

## Common errors

- `File not found` or `Invalid file type`: the search commands require an existing path ending in `.mv2`.
- `Vector dimension mismatch`: the file was embedded with a different model. Plain `maw find` still works because it uses lexical search.
- `OpenAI API key required`: only `maw ask` and `--embed openai` need it.
- `Network error` or `Rate limited`: lower `--concurrency` and `--rate-limit`, then retry.
