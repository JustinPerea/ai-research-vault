# Nightly AI research pipeline

This is how the daily digest in [JustinPerea/ai-research-vault](https://github.com/JustinPerea/ai-research-vault) is produced, and how the public page and podcast are published after it. A downstream job reads `inbox/<date>.md` at 2:45 am ET and scores papers at about 3:07 am ET, so the vault file is the hard deadline.

Times are America/New_York and follow daylight saving. This file is updated whenever the routine changes.

## Schedule

| Routine | Cron | When it runs |
|---|---|---|
| Nightly AI research vault | `CRON_TZ=America/New_York 0 21 * * *` | Every night, weekends included. The server may start it up to 15 minutes late (so about 9:00–9:15 pm ET). Digest date `D` is always the next calendar day. |
| Digest deadline guard | `CRON_TZ=America/New_York 15 1 * * *` | 1:15 am ET every night. If `inbox/<today>.md` is missing, it commits that file (a full digest if arXiv has the listing, otherwise a stub) before 2:30 am. Vault file only; no site page, no podcast. |
| Morning catch-up | weekdays around 8:00 am ET | Creates an inbox file only when it is missing entirely, and finishes the site page and podcast for a real digest that lacks them. It never rewrites an existing inbox, stub or full. |
| 8 pm overnight precheck | daily, around 8:00 pm ET | Checks only that GitHub and Gemini Notebook are signed in, and whether last night's audio shipped. It does not write a digest. |

**Guarantee:** `inbox/D.md` is on `main` by 2:30 am ET on day `D`.

The vault commit lands first. The site page and audio come later and are soft-fail: a missing page or podcast does not hold up the inbox file.

### When a run usually finishes

Measured from git on recent full runs under the previous ~2:00 am ET nightly (through 2026-11-13):

- The vault inbox usually landed at **2:18–2:25 am ET** (2026-11-06 was 2:44 am ET).
- The personal-profile digest commit and the `JustinPerea/research-audio` release usually followed **14–25 minutes later**, around **2:33–2:47 am ET** (2026-11-06 was 3:09 am ET).

Catch-up and backfill finishes are later and are not typical: 2026-11-10 vault at 8:07 am, site at 8:09 am, audio at 8:25 am ET; 2026-11-03 catch-up vault at 4:45 am, site and audio at 5:04 am ET; 2026-11-13 backfill vault at 12:01 pm, site and audio at 12:26 pm ET.

No run has finished yet under the 9:00 pm ET schedule, so a typical finish time for that schedule is not recorded.

## Steps, in order

Cloud-only, against `JustinPerea/ai-research-vault`. The run does not invent URLs, arXiv IDs, numbers, or quotes. If `inbox/D.md` already exists on `main` (stub or full), the nightly and the guard skip it.

### 1. Fetch arXiv cs.AI and the news feeds, in parallel

The candidate paper list comes from the official cs.AI listing, not from a `submittedDate` export query:

- `https://arxiv.org/list/cs.AI/recent` (one-day view: `https://arxiv.org/list/cs.AI/new`)
- One announcement day: the listing arXiv announces at 8:00 pm ET (Sunday–Thursday evenings), dated the next calendar day. At 9:00 pm ET a heading dated tomorrow is the current listing, not a missing one.
- That day-block is **new submissions plus cross-lists into cs.AI**. Replacements are skipped.
- The default URL shows only the first 50 entries. The run paginates until it has every entry in that one day-block, and does not ingest the next day's heading.
- Every kept id is verified from its abs page or the export API record. Titles are copied from the source.

**Always fetch fresh.** arXiv's list pages are behind a CDN that can serve copies hours old (this caused false "no new cs.AI day" stubs on 2026-11-10, 2026-11-12, and 2026-11-13). Every `/new`, `/recent`, and `/catchup` fetch adds a unique cache-busting query parameter (for example `&_=<unix seconds>`) and a `Cache-Control: no-cache` header, and logs the `Age` header in Sources checked. If the heading is older than expected, the run refetches with a new cache-buster a few times before concluding anything.

Hugging Face Daily Papers is allowed only as a second arXiv list. Each hit must still appear on that day's cs.AI listing and pass the stack filter.

### 2. Stack-filter and rank

The listing is the universe. The digest is not every paper on that page.

A paper is kept only if the title and abstract are about building or evaluating **agent systems** in this lane:

- Personal / iOS / on-device agents
- Multi-agent / harness / skills / MCP / tools / computer-use / GUI agents
- Memory, context assembly, KV/cache, persistence, identity across runtimes
- Local / open-weight models aimed at that stack
- Runtime guards, tool-use, long-horizon agents when they clearly serve the above

Dropped: protein/bio, medical, climate/ag, quantum hardware, pure theory, generic LLM eval, vision-only, recsys, finance-specific, and anything that is “an LLM was used” without an agent/memory/tool system.

Judgment is from title plus abstract, not title keywords alone. When unsure, the paper is kept and marked `track`.

There is no numeric score in the current skill, and **alphaXiv is not used as a rank assist**. After the keep/drop decision, each kept item gets an action signal: `try-now`, `compare`, or `track`.

Target size is a readable day, roughly 15–40 items. If more than about 40 still pass, all that pass are kept (no cap). If fewer than about 8 pass, borderline agent/memory/tool abstracts are re-read.

Listing total vs kept vs dropped is logged in Sources checked.

### 3. News and changelogs

Run first, in parallel with the arXiv fetch, by:

```
python3 /home/box/agent-data/workflows/daily-research-vault-digest/news_changelogs.py \
  --state <vault-clone>/_meta/news-state.json --write-state --out /workspace/nightly-D/news.json
```

Fixed sources, nothing else (17 feeds and pages, as named in the helper):

| Source key | What is fetched |
|---|---|
| `anthropic-api-release-notes` | Anthropic API release notes (`platform.claude.com` RSS) |
| `anthropic-news` | Anthropic news page |
| `claude-code-releases` | GitHub releases for `anthropics/claude-code` |
| `openai-api-changelog` | OpenAI API changelog |
| `openai-news` | OpenAI news RSS |
| `gemini-api-changelog` | Gemini API changelog |
| `apple-developer-news` | Apple Developer news RSS |
| `openrouter-blog` | OpenRouter blog RSS |
| `vercel-changelog` | Vercel atom feed, filtered to `/changelog/` links |
| `supabase-changelog` | Supabase changelog page |
| `mcp-spec-releases` | GitHub releases for `modelcontextprotocol/modelcontextprotocol` |
| `mem0-releases` | GitHub releases for `mem0ai/mem0` |
| `letta-releases` | GitHub releases for `letta-ai/letta` |
| `graphiti-releases` | GitHub releases for `getzep/graphiti` |
| `composio-releases` | GitHub releases for `ComposioHQ/composio` (prereleases and example packages already filtered) |
| `composio-changelog` | Composio changelog page |
| `langchain-blog` | LangChain blog RSS |

The state file is `_meta/news-state.json`. It holds `last_run` (UTC fetch time of the last published digest), `previous_run`, and `seen` item ids. The window is since the previous digest that ran (full or stub), so nothing is lost or repeated across weekends, holidays, and stub nights. Date-only sources look back one extra day (`--day-lookback` defaults to 1 when `seen` is already populated) and rely on `seen` to avoid repeats. Seen ids older than 60 days are pruned when state is written.

If the state file is missing, the helper is passed `--since` set to the UTC commit time of the newest `inbox/*.md` on `main`, together with `--state` and `--write-state`.

Customer stories, case studies, events, and hiring posts are skipped. Apple App Review Guidelines changes and any vendor AI, privacy, or data notice are always kept. A source error is not fatal.

### 4. Write `inbox/D.md`

YAML front matter, as on recent files such as `inbox/2026-11-11.md`:

```yaml
date: YYYY-MM-DD
type: research-digest
tags: [ai-research, daily-digest]
sources_checked: [arxiv-cs.AI-recent, huggingface-daily-papers, news-changelogs]
items_count: N
news_count: K
listing_count: M
conflicts_flagged: N
```

Then these sections, in order:

- `# AI Research Digest — <Weekday, Month D, YYYY>`
- `## TL;DR` — one dense bullet per item that most changes how Justin should build. The rest still appear under Items.
- `## Items` as `### N. Title` with: Source, Link, Authors/org, Focus area, What it is, Why it matters for Justin's agents, Action signal (`try-now` / `compare` / `track`)
- `## Local-model corner` — only when an included paper is a real open-weight drop; otherwise omitted
- `## News and changelogs` — required on every digest, including empty stubs. Bullets, newest first. `news_count` is the number of bullets (0 for the empty-day line).
- `## Conflicts & open questions` — required
- `## Sources checked` — listing totals (new vs cross-list vs replacements skipped), kept vs dropped, Hugging Face extras, fetch failures, and one line for news (window start, items per source, errors)

Kept paper ids are prepended onto `_meta/processed.json` and the claims ledger. News items are recorded only in `_meta/news-state.json`.

### 5. One vault commit

A single commit on `main` of `inbox/D.md` plus `_meta/news-state.json` (and, for a full digest, the updated `_meta/processed.json` and `_meta/claims-ledger.md`). Message form: `digest: YYYY-MM-DD — <short headlines> (#N / listing M)`.

Each inbox file is committed once. A stub is not later replaced by a full digest.

### 6. Public site page

After the vault file is on `main`, the public page is built in a throwaway clone of `JustinPerea/personal-profile` (no push yet):

- `https://justinperea.com/research/D` from `public/research/D.md`, plus the machine twin `public/research/D-llms.md`, `index.json`, sitemap/feed, and images under `public/research/images/D/`.
- Papers first, then the vault's News and changelogs section (same items and links, rewritten in public voice). No news beyond that section, and no announcement art.
- One real figure per paper, via the extract-arxiv-figure helper (`extract_arxiv_figure.py`). It tries the arXiv HTML page, then ar5iv, then the PDF. If it finds no usable figure, that paper's figure is omitted; the paper stays. Figures are never fabricated.

This step is skipped on stubs (`items_count: 0`).

### 7. Podcast

See [Podcast](#podcast). Also skipped on stubs. Soft-fail: a podcast failure does not block the vault or the site page.

### 8. One personal-profile commit, deploy, smoke check

After audio (or after a skipped/failed podcast), one commit on `JustinPerea/personal-profile` of the site page plus the patched audio URLs. The run waits for the Vercel production deploy of that commit to reach `READY`, then runs:

```
python3 /home/box/agent-data/workflows/public-research-site-digest/smoke_check.py YYYY-MM-DD
```

The smoke check confirms the live index row, basename-only `file`, news count, paper list, figures, images, and (if claimed) that audio URLs are not HTML. The page is not treated as live if smoke is red.

### 9. Mac-side pull into Obsidian

On Justin's Mac, a 7:00 am ET job pulls the vault, and a 7:20 am ET launchd job `com.justinperea.ai-research-auto-promote` runs `/promote-digest` into Obsidian `Research/topics`. Promote commits typically land on this repo around 7:22–7:25 am ET (for example `007834f` at 7:22 am ET on 2026-11-15, for the 2026-11-13 items).

## Podcast

Generated only for a real (non-stub) digest, and only after the site files exist in the personal-profile clone.

**What is sent.** arXiv abs URLs only, taken from that day's `public/research/D-llms.md` if it exists, otherwise from the vault inbox. Capped at 45 (the NotebookLM source limit). If the vault has more than 45 papers, 45 abs URLs biased to the same stack (personal/iOS agents, multi-agent, memory/context, computer-use, MCP/tools, local/open models) are sent; the vault and the site page still list every paper. News URLs are never added.

**How it is sent.** There is no consumer API. A browser-driving helper on the cloud computer uses Justin's already signed-in Google session (Pro) at `notebook.google.com` or `notebooklm.google.com`. If a login wall appears, the step stops; the password and 2FA are never typed.

1. Create a **new** Gemini Notebook every day (never reuse). Title exactly: `AI Research Digest — YYYY-MM-DD` (em dash).
2. Add sources via Websites, pasting the newline-separated abs URLs in one batch.
3. Studio → Audio Overview. There is no customize step and no extra focus or prompt.
4. Wait until it finishes (often several minutes). Download the source `.m4a`. Grab a transcript if the UI offers one.
5. Encode `YYYY-MM-DD.opus` from the m4a (48 kbps opus is fine). Keep the m4a as AAC fallback.

**Where the audio goes.** Binaries are not committed to `personal-profile`. They are uploaded to a public GitHub Release on `JustinPerea/research-audio` whose tag and name are `D`, with assets `D.m4a` and `D.opus` (and a transcript file when one was offered). That date's existing `index.json` row is then patched with:

```
audio: {
  url: "https://github.com/JustinPerea/research-audio/releases/download/D/D.opus",
  duration: <seconds>,
  m4a_url: "https://github.com/JustinPerea/research-audio/releases/download/D/D.m4a"
}
```

A podcast failure is reported as `Audio: no - <reason>`. Login wall, quota, or UI breakage skips audio and does not block the vault or the site digest. Audio is not recapped as shipped until a curl of the release asset is not HTML.

## Weekends, holidays, and failures

- **Friday and Saturday evening runs, and arXiv holidays:** commit a stub with `items_count: 0` and the News and changelogs section. Skip the site page and the podcast.
- **Late arXiv mailing (Sunday–Thursday evening):** the 9:00 pm run commits nothing. The 1:15 am guard commits `inbox/D.md` (full if the listing has appeared, otherwise a stub) before 2:30 am.
- **One commit per inbox.** Never commit a stub first and replace it with the full digest later. (This was the failure mode on 2026-11-10 and 2026-11-13.)
- **Vault is the hard gate.** Site and podcast are soft-fail.
- **Podcast failure:** reported as `Audio: no - <reason>`.
- **Two GitHub auth failures in a row** pause the routine.
- **Stale CDN pages:** refetch with a new cache-buster, as in step 1. Do not publish a false empty stub from a stale heading.
- If the digest is not published, the vault clone is discarded so the next run re-covers the news window.

## Last 7 runs

Times are America/New_York. Start times are given only where they are known; otherwise "not recorded." Finish time is the vault inbox commit (that is when `inbox/D.md` lands on `main`). Site and audio commits, when they exist, are noted in the result.

Before 2026-11-13 the nightly ran at about 2:00 am ET. On 2026-11-13 the schedule was changed to Sunday–Thursday 8:30 pm ET, so Friday evening (which would have written 2026-11-14) and Saturday evening (which would have written 2026-11-15) did not run. Those two dates have no file.

| Digest date | Start (ET) | Finish (ET) | Result | Commits |
|---|---|---|---|---|
| 2026-11-15 | no run | — | none (no file) | — |
| 2026-11-14 | no run | — | none (no file) | — |
| 2026-11-13 | about 2:17 am; backfill start not recorded | stub 2:18:24 am; full backfill 12:01:40 pm | stub first (stale arXiv CDN page, `listing_count: 0`), then a full backfill approved by Justin (39 / 321), with site and audio at 12:26 pm | vault stub `1ec9354`, news_count fix `4546941`; full vault `95425da`; personal-profile `c4ac950`; audio release `2026-11-13` |
| 2026-11-12 | about 2:00 am | 2:22:21 am | stub (`items_count: 0`, news 9; false "no new cs.AI day" from a stale CDN page). No site, no audio | vault `77e5e2f` |
| 2026-11-11 | about 2:00 am | 2:25:38 am | full (34 / 248). Site `f6f81ec` at 2:39:15 am; audio release `2026-11-11` at 2:39:01 am | vault `9a8a30f`; personal-profile `f6f81ec` |
| 2026-11-10 | about 2:00 am (stub); morning catch-up about 8:00 am | stub 2:24:20 am; full 8:07:22 am | stub first (stale CDN), then morning catch-up full (37 / 480). Site `2d908bf` at 8:09:37 am; audio added `865539c` / release `2026-11-10` at 8:25 am | vault stub `6aae4f8`; full vault `89b9e34`; personal-profile `2d908bf`, `865539c` |
| 2026-11-09 | about 2:00 am | 2:24:46 am | full (34 / 220). Audio release `2026-11-09` at 2:42:25 am; site `0dff85a` at 2:44:46 am | vault `edb0f84`; personal-profile `0dff85a` |

## Change log

- **2026-11-15.** Nightly moved to 9:00 pm ET every night, including weekends (`CRON_TZ=America/New_York 0 21 * * *`). Added a 1:15 am ET deadline guard (`CRON_TZ=America/New_York 15 1 * * *`) so `inbox/D.md` is on `main` by 2:30 am ET. Friday and Saturday evenings, and arXiv holidays, commit a stub with `items_count: 0` and the news section. Each inbox is committed once; a stub is not later replaced by a full digest. Morning catch-up never rewrites an existing inbox. This file added.
- **2026-11-13.** Nightly moved to Sunday–Thursday 8:30 pm ET at Justin's request. Digest named for the next-day listing date (a Thursday-evening run writes Friday's file). arXiv fetches are cache-busted, with the `Age` header logged, after false empty stubs from a stale CDN. That Sunday–Thursday window is why 2026-11-14 and 2026-11-15 were never written.
- **2026-10-17.** News and changelogs section first appeared in the vault, with `_meta/news-state.json` (`6dcb584`, weekend stub `inbox/2026-10-17.md`). The 2026-10-16 digest has no news section.

The date the digest was switched to a one-announcement-day, cs.AI-only listing is not recorded in this repo. That is the current rule in the vault skill.

This file is updated whenever the routine changes.
