---
name: ingest-resource
description: Ingest an external resource into Pierre's knowledge base — full read/write on Claude Code/Mac (local files, auto-committed+pushed by a Stop hook), git-based read/write on a Claude Code Cloud session (clone claude-kb directly, commit+push manually — no YouTube fetch there, network gateway blocks it), chat-only synthesis on claude.ai/mobile (no write path exists there; Pierre must attach KB files manually via "Add from GitHub" and copy the final entry into the repo himself). Takes a URL or pasted content, reads the destination context, forms a view on what it means for Pierre, discusses it with him, and only then drafts a compact entry routed via the live index at knowledge/index.md, formatted to match that file's existing entries. Asks for confirmation before writing anything (Mac/Cloud) or before finalizing draft text (mobile). Use whenever Pierre shares a link or transcript he wants kept — "add this to my knowledge base", "ingest this", "save this for later", "this belongs in tools" — even if he doesn't name the skill.
argument-hint: [url — or run bare and paste the content]
---

# Ingest Resource

Move the useful part of an external resource into the knowledge base — and only the useful part. A 40-minute transcript that yields four bullets is a success; a full summary pasted into the knowledge base is a failure. **Never write to any file before Pierre approves the exact text** — that confirmation is the one hard rule of this skill, in every environment.

This skill produces two different outputs, not one: a **synthesis** discussed with Pierre in chat (the qualitative read — thesis, confidence, what it actually means for him, grounded in what's already on record) and a **compact entry** that gets written to the file (the archival record). Don't collapse them. The chat synthesis is conversational and can run long. The file entry stays terse and matches the target file's existing style, regardless of how long the discussion ran.

## 0. Which environment is this?

This skill runs in three places with different capabilities. Don't ask Pierre which one you're on — detect it:

1. Try a local `Read` on `~/.claude/knowledge/index.md`. Succeeds → **Mac**, use local tools for everything below.
2. That fails, but a `Bash` tool exists → **Claude Code Cloud session** (started remotely, e.g. from Pierre's phone). No personal files are present (skills, scripts, `~/.claude/knowledge` all live only on the Mac's disk), but you have real git/network access — see the Cloud flow below.
3. No filesystem and no `Bash` tool at all → **claude.ai mobile chat app**. See the Mobile flow below.

- **Mac (Claude Code)** — full local filesystem access. Read/Write/Bash reach `~/.claude/knowledge/` directly, including `git commit`/`push` (handled automatically by a Stop hook, `~/.claude/scripts/kb-autosync.sh` — you don't need to commit/push yourself, just write the file).
- **Claude Code Cloud session** — a real Bash/git environment, but an ephemeral sandbox with none of Pierre's personal files. It can `git clone`/push to GitHub directly (confirmed working), giving genuine read+write — better than mobile's dead end. But its network gateway is locked to a curated allowlist (package registries, GitHub, cloud SDKs) that does **not** include YouTube, confirmed by direct testing — `yt-dlp`/raw fetches to YouTube get rejected at the gateway, and there's no unlock from inside the sandbox. SocialKit MCP calls are a separate channel from raw sandboxed network calls and may still work — worth trying, not guaranteed. See the Cloud flow below.
- **Mobile (claude.ai chat app)** — no local filesystem, and **no write path to the repo at all.** The "GitHub Integration" connector visible in claude.ai's settings is a manual file-attach feature ("Add from GitHub" in the message compose box), not a tool-calling connector — confirmed by direct testing. There is no tool Claude can call on mobile to read, write, or commit anything in `claude-kb`. Don't search for GitHub file-read/file-update tools on mobile; they don't exist, and searching for them wastes a turn.

### Cloud flow (real git access, no personal files, no YouTube)

1. Check for an existing `claude-kb` checkout in the working directory first. In the normal case — a Cloud session started against the `claude-kb` repo — the harness pre-clones it before you even launch, so this step is usually already done; you're checking, not expecting to need step 2. Its `origin` remote may be a local authenticating proxy URL (`http://local_proxy@127.0.0.1:.../git/...`) rather than the raw `https://github.com/...` one — that's normal, functionally identical, push lands on real GitHub regardless. If a checkout exists, `git fetch && git merge --ff-only origin/main` (or equivalent) to bring it current before reading anything — don't assume a checkout you didn't just create is up to date. Only if none exists at all: `git clone https://github.com/pierreboulanger/claude-kb.git`.
2. Once cloned/synced, treat that checkout exactly like `~/.claude/knowledge` for the rest of this skill — §2 (route), §3 (read destination), §4 (synthesize) all apply unmodified against it.
3. For §1 (fetch): the Mac-only methods (local script, local SocialKit skills needing a local API key) aren't available, and neither is any method that requires raw network access to a non-allowlisted domain. Try SocialKit MCP for platforms it covers; for everything else, ask Pierre to paste — same as the mobile column in §1's table.
4. At what would be §7: there **is** a real write step here, unlike mobile. After Pierre approves the exact text, `Write`/`Edit` the file, then `git add`/`commit`/`push origin main` yourself — **push straight to `main`, don't open a PR and stop there.** Confirmed by direct testing: a raw push to `main` is not blocked by the sandbox (Anthropic's own docs suggest git pushes are restricted to the session's current working branch, but that did not hold up in practice — a direct `git push origin <branch>:main` succeeded with no proxy rejection and no PR object created). If the session's task setup has pinned it to a working branch by default (normal for coding tasks), push to `main` anyway rather than leaving the commit stranded on that branch awaiting review — the content already went through its one real review at the §7 approval step, before the commit was even made. A PR here would just be re-approving the same text a second time, and would leave the entry not-actually-saved until Pierre does a separate manual merge — exactly the friction this flow exists to avoid.

   Two things confirmed by direct testing (2026-07-16), worth trusting over general git-push instinct: **(a)** `git push origin <working-branch>:main` succeeds as described above, but **does not** update the working branch's own `origin/<branch>` ref — that ref is left stale, pointing at wherever it was before. **(b)** The Stop hook (`~/.claude/stop-hook-git-check.sh`, see below) compares local commits against `origin/<current-branch>`, so a stale branch ref makes it falsely flag already-pushed, already-safe commits as unpushed or unverified — including commits that aren't even this session's own (e.g. a Mac session's commits pulled in via a rebase onto `origin/main`). That's not a hook bug and not something to fix by editing the hook — **fix it by keeping the branch ref honest**: immediately after every push to `main`, also run a plain `git push origin <branch>` (no refspec) in the same breath, so the branch's remote ref fast-forwards to match. Do this every time, not just when the hook complains.

   If the push to `main` is rejected because something else pushed to `main` in the meantime (e.g. a concurrent Mac session) — confirmed workable: `git fetch origin main && git rebase origin/main`, then push both `main` and the branch again. This resolves cleanly when there's no actual file-level conflict; if there is one, resolve it normally before pushing.

   There **is** a Stop hook in this environment (`~/.claude/stop-hook-git-check.sh`), but don't mistake it for Mac's autosync — it only checks/nags on git state (uncommitted changes, unpushed commits, unverified commit authorship) and never commits or pushes anything itself. Its "Stop hook feedback" messages are a checker complaining, not a confirmation that anything synced — the actual `git add`/`commit`/`push` is still entirely on you. If it flags something as unpushed/unverified, check whether the branch ref is just stale (per above) before assuming the hook's suggested fix (e.g. rewriting commit authorship) is correct — it frequently isn't: don't blindly amend/rebase authorship on commits that aren't yours (a Mac session's own commits, correctly attributed to Pierre, are not a problem to "fix"). If the push fails outright (auth, network), say so plainly: this sandbox is ephemeral, and a commit that only exists locally here is lost once the session ends — it is not durably saved until the push actually succeeds.
5. Mac-only files (health.md, portfolio-manager/ except resources-finance.md) are excluded from the GitHub repo the same way they're excluded from mobile — see §2a, which applies here too, not just to mobile.

### Mobile flow (no write path — chat-only, Pierre does the write)

1. If Pierre hasn't already attached KB files for context, ask him to use "Add from GitHub" to attach `index.md` and the likely target file(s) before you route/synthesize — you can't read the repo any other way. If he hasn't attached anything and can't yet, do the fetch/synthesis (§1, §4) anyway and note the routing/grounding is provisional until he attaches the destination file.
2. Do §1 (fetch), §3-6 (read context from whatever was attached, synthesize, discuss, draft) exactly as on Mac — none of that requires writing.
3. At what would be §7, there is no write step. Give Pierre the exact final entry text and target file, and tell him plainly he needs to add it himself — either by pasting it into GitHub's own web/mobile file editor, or by holding onto it (or asking you to repeat it) for his next Mac session. Do not say it's "queued" or "saved" — it isn't, until he does that manually.

**Mac-only files on mobile or Cloud:** the repo excludes `health.md` and everything in `portfolio-manager/` except `resources-finance.md` — real financial-position and health data that intentionally never leaves the Mac. `index.md` marks exactly which files are repo-synced vs. Mac-only. If step 2 below determines a Mac-only file is the right target, don't draft repo content for it outside the Mac — just show Pierre the synthesis and tell him to capture it himself next time he's on Mac (there is no automated queue — see §2a).

## 1. Get the content

Detect the source type from the URL and apply the matching method — check this matrix top to bottom before falling back to anything generic. Two access paths to SocialKit exist and are **not interchangeable**: the hosted **MCP connector** (works on Mac, mobile, and likely Cloud — untested there — but only reaches YouTube/TikTok/Instagram/Facebook) and the local **SocialKit skills** (reach all six platforms including X/LinkedIn, but only run on Mac — they shell out to `curl` and read `SOCIALKIT_API_KEY` from the Mac's own environment, which exists on neither mobile nor Cloud). Use MCP wherever it covers the platform; fall back to skills only for the two platforms MCP can't reach, and only on Mac.

| Source | Mac method | Cloud method | Mobile method |
|---|---|---|---|
| YouTube | `~/.claude/scripts/yt-transcript "<url>"` via Bash (local, free — try this before MCP, since it costs no credit) | SocialKit MCP — **confirmed working** (`youtube_stats`/`youtube_transcript` pulled successfully in this environment). **Never try a raw fetch/`yt-dlp` call here**: the network gateway blocks YouTube by default and there's no unlock from inside the sandbox, confirmed by direct testing (recentRelayFailures log on the proxy) | SocialKit MCP (look up the current transcript tool via `ToolSearch` once the connector is loaded — don't hardcode a tool name here, it may change) |
| TikTok | SocialKit MCP | SocialKit MCP (untested — try it) | SocialKit MCP |
| Instagram | SocialKit MCP | SocialKit MCP (untested — try it) | SocialKit MCP |
| Facebook | SocialKit MCP | SocialKit MCP (untested — try it) | SocialKit MCP |
| X / Twitter | SocialKit skill (`engagement-analysis` for a single tweet, `channel-research` for a profile/recent-tweets list) — **Mac only, MCP has no X coverage** | Ask Pierre to paste — no automated path exists | Ask Pierre to paste — no automated path exists on mobile |
| LinkedIn | SocialKit skill (`engagement-analysis` for a single post, `video-transcripts` for a video post) — **Mac only, MCP has no LinkedIn coverage** | Ask Pierre to paste — no automated path exists | Ask Pierre to paste — no automated path exists on mobile |
| Reddit | Claude in Chrome extension → paste the content (WebFetch of `.json` is blocked at the gateway as of 2026-07-19; retest before assuming) | Ask Pierre to paste — no Chrome extension in the sandbox | same |
| Threads | Ask Pierre to paste — no automated method exists | same | same |
| Podcast (non-YouTube, e.g. Spotify/Apple) | Ask Pierre to paste — not covered by SocialKit | same | same |
| Web article / docs page | WebFetch directly | WebFetch directly (likely works — routed differently from a raw sandboxed network call, but not directly confirmed) | same |
| PDF | Read from the project folder Pierre points to | Ask Pierre to paste the extracted text | Ask Pierre to paste the extracted text — no guaranteed file access on mobile |
| Pasted content (no URL) | Process directly, no fetch | same | same |

**On X/Twitter threads specifically** (a multi-post chain, not a single tweet): neither MCP nor the skills can assemble one — SocialKit's API has no thread-assembly endpoint on any access path, confirmed by direct testing. `engagement-analysis` returns one tweet at a time with no reply-chain field. If Pierre wants a thread ingested, ask him to paste it — this isn't an environment limitation, it's a vendor API gap that applies everywhere.

- Fetch fails, hits a paywall, or returns an obvious teaser → say what happened and ask Pierre to paste the full text. Don't extract from a snippet as if it were the whole article.
- YouTube link errors (no captions, network) → tell Pierre what happened and ask him to paste the transcript.
- No URL and nothing pasted → ask Pierre for the content.

## 2. Route it — figure out the destination before reading anything else

Don't rely on a fixed list of files here — the knowledge base is expected to grow and reshape over time, and a routing table baked into this skill would silently go stale the moment a file is added, split, or renamed (it already has, more than once).

Read **`knowledge/index.md`** — local `Read` on Mac; the cloned checkout on Cloud (see §0); on mobile, only from whatever Pierre attached via "Add from GitHub" (see §0) — and match the content against its topic descriptions. On Mac or Cloud, cross-check with `ls` on the knowledge directory: a file on disk with no matching index entry is a gap worth flagging in your final report, not silently ignoring. (No equivalent cross-check on mobile — you only see what Pierre attached.)

- Content spanning two files → give it one primary home and a `[[other-file]]` wiki-link there, rather than duplicating.
- `long-horizon-protocol.md` is an authored protocol document — never an ingest target, no matter what its index entry says.
- No file's description genuinely fits → propose options (including a new knowledge file) and let Pierre pick.

Do this routing decision *before* extraction, not after — §3 below needs to know which file(s) to read for context, and §4's synthesis needs to know the destination before it can say anything grounded.

This works without ever editing this skill because step 7 below writes back to `index.md` whenever routing changes — the index you read here is kept current by this same skill's own output.

### 2a. Target is a Mac-only file, and you're on mobile or Cloud

Don't attempt to write extracted content anywhere in the repo — there's no repo-synced file that's the right home, by design (these files are gitignored, so a Cloud clone doesn't have them either, and mobile has no write path regardless — see §0). Instead:

1. Show Pierre what you found — the synthesis (§4) still applies in full.
2. Tell him plainly: this needs to go into `<target file guess>` next time he's on Mac. There is no automated queue — give him the source URL and a one-line context note in chat so he (or a future Mac session, if he pastes this back) can capture it then. Don't claim it's "queued" or "saved" anywhere.

## 3. Read the destination before forming a view

Before drafting anything — before even deciding what's worth keeping — read the existing material in the destination area, not just the single file you expect to append to:

- If the target lives in a folder (e.g. `portfolio-manager/`), read every file in that folder you have access to, not only the one you're about to write into. On Mac, that's the full folder (`snapshot.md`, `profile.md`, `cashflow.md`, `resources-finance.md`, `change-log.md`). On mobile or Cloud, it's whatever's repo-synced — see §0's limitations for each.
- If the target is a standalone file (`tools.md`, `business.md`, `product-playbooks.md`, `health.md`), read that file in full.
- This applies to every domain, not just investments. If a domain later grows into a folder with several files (e.g. health splits into diet.md / labs.md / protocols.md), read all of them the same way — the rule is "read the destination area," not "read portfolio-manager specifically."

The point: you can't tell Pierre what a new resource means for him without knowing what he already has on record. Skipping this is how ingestion produces a generic "Relevance: touches SPYI" line instead of one that cites his actual €431k cash position, or a health entry that misses that the new supplement contradicts a contraindication already logged.

## 4. Synthesize

Form a view before extracting bullets. This happens in chat, not in the file.

1. **Thesis** — 2-3 sentences: what is this resource actually arguing or reporting, at the top level. Not a summary of topics covered — the actual claim.
2. **Confidence-tag the conclusions**, not the individual facts — use Pierre's own framework from his global communication rules: [Certain] / [Likely] / [Guessing]. A claim can be [Certain] that the source said it while the underlying thesis is [Guessing]. Tag the thesis-level claims, not "did the speaker say X" — that distinction is the whole point of tagging.
3. **Push back where warranted** — if a claim is overstated, contradicted by something already in the destination files you just read, or rests on an assumption that doesn't hold, say so. Don't relay a thesis neutrally if it doesn't hold up. This is the "advisor, not assistant" posture from Pierre's global rules — accuracy over agreement, applies here too.
4. **What this means for Pierre** — grounded in §3, not generic. Cite actual tickers, EUR amounts, existing positions, protocols, or frameworks already on record. If the resource changes nothing he's already decided, say that explicitly rather than manufacturing a connection.

Apply the per-domain bar below when deciding what's even worth this treatment — if nothing clears the bar, tell Pierre the resource had nothing durable for the knowledge base and stop. Don't force a synthesis on material that doesn't warrant one.

### Per-domain bar

Once it's clear which file this is headed for, apply that file's specific bar on top of the general one above — this is what keeps entries sharp instead of generic:

- **`portfolio-manager/` (investments)** — Mac-only except `resources-finance.md`. Keep specific figures with their date/context, a thesis *with* its reasoning, tax/regulatory facts, and clear confidence framing. Drop generic market commentary, price predictions with no stated mechanism, hype. Route within the folder (Mac only): current portfolio figures → `snapshot.md` (+ dated line in `change-log.md`); thesis/tax/framework facts → `profile.md`; net-worth or cash-flow readings → `cashflow.md`; macro-event analyses and research-source assessments → `resources-finance.md` (repo-synced — this is the one mobile can actually reach). Never write history into `snapshot.md`.
- **`business.md`** (CRO / EPAM work) — keep test methodology, sample sizes, effect sizes/conversion rates, named frameworks (ICE, Fogg, Cialdini, etc.) and tools, and results — flagged "single data point" when the source doesn't establish it as a real benchmark. Drop generic "always A/B test" advice and conversion-rate claims with no sample size attached.
- **`health.md`** — Mac-only. Keep specific products, doses, protocols, mechanisms, and contraindications/caveats. Drop generic wellness advice not tied to a specific recommendation. On mobile or Cloud, see §2a — capture only, never extract.
- **`product-playbooks.md`** — keep concrete stack/tool choices with pricing tiers and upgrade thresholds, workflow steps, build-decision criteria. Drop generic productivity advice not tied to a specific stack decision.
- **`tools.md`** — keep pricing, limits, benchmark numbers, licensing, named caveats/failure modes. Drop tool mentions with no concrete detail.
- **`kb-system.md`** (the knowledge base itself) — the routing home for RAG / memory-system / knowledge-tooling resources (LLM wikis, agent memory, retrieval patterns), which previously defaulted to `tools.md`. Keep adoption criteria and thresholds, measured limits, failure modes, and concrete workflow mechanics; drop generic "second brain" productivity hype. Ingested entries land under `## Patterns & tools evaluated` or `## Ingestion stack` only — the `## Decisions` (append-only, dated) and `## Recall misses` sections are maintained by working sessions, never by ingestion.

No bar listed above for a file → apply the general bar only; don't invent domain criteria for files not covered here.

## 5. Discuss or draft — ask Pierre

Present the §4 synthesis in chat, then ask directly: does he want to debate or refine it, or should you draft the KB entry now?

- **Debate** → this is a normal conversation, not a scripted branch — follow wherever it goes. When Pierre signals he's ready ("ok, draft it" / "log it" / similar), move to §6 carrying whatever the discussion changed: updated thesis, dropped claims, added context, a different confidence call.
- **Draft now** → go straight to §6 with the synthesis as-is.

Don't skip straight to drafting without asking, even when the synthesis seems obviously final. Asking is what turns ingestion into an exchange instead of a one-way dump.

## 6. Format to match the target file

Read the target file before drafting, if you haven't already in §3, and mirror its heading level, bold-label bullet style, and `---` separators. For tool-like entries, `tools.md` sets the house pattern:

```markdown
## <Name>
- **What:** one line
- **Why useful:** the payoff, with numbers if the source gave them
- **Caveat:** limits or failure modes
- **Relevance:** which of Pierre's projects this touches
- **Source:** <url or "pasted transcript"> — added <YYYY-MM-DD>
```

Keep this entry terse regardless of how long the §4-5 discussion ran — the file entry is the archival record, not a transcript of the conversation. Use the destination file's own existing tag convention here (e.g. `resources-finance.md`'s [Explicit]/[Likely]/[Unverified]), which may differ from the [Certain]/[Likely]/[Guessing] used in the chat synthesis — that's expected, not a bug: chat uses Pierre's global vocabulary, the file uses whatever convention its existing entries already established.

Whatever the shape, every entry ends with a source + absolute date line (today's real date, never "today"). If the resource contradicts or updates an existing entry, draft an in-place update to that entry and state explicitly what changes — never leave two conflicting entries in the file.

## 7. Confirm, then write (Mac and Cloud — see §0 for mobile)

Before touching any file, show Pierre:

1. The target file, and whether this is an append or an in-place update
2. The exact entry text, in full
3. Any one-line hook update this implies for `knowledge/index.md`

Ask with AskUserQuestion — options like "Write it", "Different file", "Trim it", "Skip". Apply whatever he picks and re-confirm if the text changed materially. Only after approval, write with local `Write`/`Edit`, then:

- **On Mac:** tell Pierre the entry landed in the file — the `kb-autosync.sh` Stop hook commits and pushes it to `origin/main` automatically as soon as the current response ends (Stop hooks fire after every completed response, not at session end) and reports what it did in the transcript; no manual git step needed.
- **On Cloud:** `git add`/`commit`/`push origin main` yourself, right after writing. See §0's Cloud flow for the full mechanics (push straight to `main`, keep the branch ref in sync, what to do if the push is rejected, and the Stop hook's actual behavior) — followed here rather than repeated, so there's one source of truth instead of two that can drift out of sync.

On mobile, there is no write step — see the mobile flow in §0 instead.
