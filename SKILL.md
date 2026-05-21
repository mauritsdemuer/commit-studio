---
name: commit-studio
description: Turn a git repo's commit history into a portfolio artifact in the user's voice — long-form logbook (book), social posts and carousels (LinkedIn, Instagram, X), conference talk script, personal journal, or a custom format the user describes. Asks once what they're making, scores commits for fit against that format, writes a calibration draft so the user can confirm voice/angle before bursting through the rest, then drafts and runs short interviews to preserve their voice. Detects user identity across multiple commit emails, treats AI-tool commits the user drove (v0, lovable, bolt, cursor-agent, copilot) as theirs, survives rebases, handles update runs when new commits land, and for social/talk formats offers a visual companion path (hand off to an image skill or output paste-ready prompts for ChatGPT / Nano Banana / Midjourney). Use when the user asks for a portfolio writeup, narrative changelog, dev logbook, "a book from my commits", "tell the story of what I built", a LinkedIn carousel from their work, a talk script, a personal dev journal, or any variant of turning commit history into a finished artifact.
---

# Commit studio

Produce a finished artifact from the invoking user's git commit history. The artifact varies — long-form book, social posts, talk script, personal journal, custom — but the workflow is the same: detect who the user is, dump the corpus, propose a structure, write a calibration draft, confirm direction, then burst through the rest. Pair the technical narrative with short interviews that preserve the user's voice without fabrication.

The user invokes this from inside a git repo (or points to one). Treat them as a smart collaborator who hasn't read this file. One sentence of orientation up front, then drive.

---

## The target output, in one chapter excerpt

Use this as the prose-voice calibration target. Every chapter you write should land at this density, this tone, this honesty:

```
## Chapter 14. The avatars that vanished on every deploy

Cloud Run replaces containers on every deploy. The Rails app had
been writing Active Storage uploads to the container's local
`storage/` directory the whole time. Every avatar uploaded since
the last image rolled out was silently gone the next time we shipped.

The mortifying part was the terraform. The GCS bucket and IAM bindings
had been provisioned in `terraform/environments/storage.tf` for
months. The bucket name was landing in the container as `GCS_BUCKET`
via `app_plain_env`. The Rails side just wasn't pointing at it.

`b971317 fix: store Active Storage uploads in GCS (Cloud Run is
ephemeral)` was three small changes: add the `google-cloud-storage`
gem, add a `google` service block in `config/storage.yml` reading
`ENV['GCS_BUCKET']` (no credentials key — Cloud Run's workload identity
handles auth via the metadata server), flip `production.rb` from
`:local` to `:google`. Zero terraform changes. Uploads from before
this change were on now-replaced containers and had to be re-uploaded
once.

### From me

> The GCS bucket had been sitting there for months. Terraform was
> right; the Rails side wasn't wired up. The bug was three lines
> from being fixed and I hadn't seen it. Felt stupid in a
> productive way.
```

Key things to copy from this:

- **Start in the work, not announcing the work.** First sentence drops the reader into the state of the codebase. No "In this chapter we will look at..."
- **Cite commits inline as evidence.** Short hash + subject in backticks. Never invent.
- **Honest about the slow part.** "The fix took twenty minutes; the decision took two days." Specifics beat generalisations.
- **Reflection is the user's exact words.** Light punctuation tidy only. Never expand a terse answer into eloquent prose.
- **No recap at the end.** When the point is made, stop.

---

## Phase 0 — Detect mode, identity, scope

Scan silently before asking anything.

### 0a — Verify it's a git repo

```bash
git rev-parse --is-inside-work-tree
```

If not a repo:

> I need a git repo to scan. Is the project cloned somewhere local? Paste the path. Or paste a GitHub/GitLab URL and I'll clone it.

If they paste a URL, `git clone` into a working dir then continue from inside.

### 0b — Check for existing logbook (mode detection)

Look for `.logbook-state.json` in (1) the default output dir `~/<repo-slug>-logbook/`, (2) the current dir, (3) any path the user mentioned.

| Found | Mode |
| --- | --- |
| State + chapters, `last_commit_hash` equals current HEAD | Offer **revise mode** for a specific chapter |
| State + chapters, N new commits since `last_commit_hash` | **Update mode** — skip kickoff, fold new commits in |
| State references commits not reachable in `git log` (rebased/squashed) | **Rebase recovery** — see section below |
| Chapters exist but state file deleted | Ask: rebuild state from chapters, start fresh, or work elsewhere? |
| Nothing found | **New run** — continue to 0c |

### 0c — Detect identity (new run only)

Run in parallel:

```bash
git config user.name
git config user.email
git shortlog -sne --all
git log --all --format='%an <%ae>' | sort -u
```

**Clustering rules — concrete, not vibes:**

| Rule | Example |
| --- | --- |
| Same name + same part-before-@ across different domains → same person | `jane@gmail.com` + `jane@work.co` |
| Plus-tag Gmail variants collapse to the canonical address | `jane+gh@gmail.com` + `jane+work@gmail.com` + `jane@gmail.com` |
| GitHub noreply `<NUMERIC_ID>+<username>@users.noreply.github.com` cluster by NUMERIC_ID | `12345+jane@users.noreply.github.com` and `12345+jane-dev@users.noreply.github.com` are the same account |
| `Co-Authored-By:` trailers in commit body — human-shaped attributions count as humans; tool-shaped (Claude Code, Copilot Agent) count as AI tools | `Co-Authored-By: Claude Opus <noreply@anthropic.com>` → AI tool |
| Anything matching `*[bot]` AND matching automation patterns (dependabot, renovate, vercel, netlify, github-actions, aikido-autofix) → automation, default exclude |  |
| Anything matching `*[bot]` AND matching AI-tool patterns (v0, lovable, bolt, cursor-agent, replit-agent, copilot-agent) → AI tool, ASK if user was driving |  |

Bucket the result. Present in **one** confirmation message that also surfaces the repo path so the user can spot a wrong-directory mistake before any work happens. Omit empty sections:

```
Scanning: /home/jane/code/myrepo (myrepo)
587 commits total, 2026-04-10 → 2026-05-20, 12 branches detected

If this isn't the repo you meant, tell me the right path and I'll switch.
Otherwise: here's what I see — tell me what's yours.

YOU (high confidence):
- Jane Smith <jane@example.com> — 210 commits
  Variant: Jane Smith <12345+jsmith@users.noreply.github.com> — 45 commits
  Variant: jane <jane@work.co> — 12 commits

SOMEONE ELSE WAS HERE TOO:
- John Doe <john@example.com> — 50 commits
  → acknowledge in prose / exclude entirely?

AI TOOLS — WERE YOU DRIVING ANY OF THESE?:
- v0[bot] — 80 commits
- cursor-agent — 30 commits

AUTOMATION (defaulting to exclude):
- dependabot[bot] — 15 commits
- vercel[bot] — 30 commits

MYSTERY (anyone you recognize?):
- unknown <user@localhost> — 5 commits
```

The repo path + commit count + date span at the top is the wrong-dir sanity check — the user can spot "wait, that's not the project I meant" in one glance before identity confirmation even starts. If they redirect, re-run 0a through 0c from the new path.

If **>30% of in-scope commits end up being AI-tool bots the user drove**, flag once before kickoff:

> Heads-up: most of this repo was built through [tool]. Commit messages from those tools tend to be terse or generic. When I write chapters I'll lean harder on diff inspection — what actually changed in the files — than on commit-message rhetoric.

**Squash-merge attribution caveat.** When a teammate's PR is squash-merged, the resulting commit on `main` is authored by whoever merged it (often the user), not the PR's original author. The skill's author filter will count those as the user's work even though the content was someone else's. If you notice a commit in the outline that looks suspiciously large or out of character (a single commit with hundreds of changed files, or a subject that reads like "Squash and merge PR #N"), surface it during Phase 2 outline review and ask: "this looks like a squash of someone else's PR — keep, drop, or attribute to the co-author?" Co-Authored-By trailers in the commit body often confirm the original author.

### 0c-bis — Branch scope (conditional)

After identity, count how many branches have **unique commits by the user** (commits not reachable from any other branch). Run:

```bash
# For each branch, count commits by the confirmed user that aren't on any other branch
for branch in $(git for-each-ref --format='%(refname:short)' refs/heads refs/remotes); do
  unique=$(git rev-list "$branch" --author='<author-regex>' --not $(git for-each-ref --format='%(refname:short)' refs/heads refs/remotes | grep -v "^$branch$") 2>/dev/null | wc -l)
  echo "$branch $unique"
done
```

If **only one branch** has unique commits (the common case — solo dev on `main`, or team where everything flows through `main`), silently use `--all` and move to 0d. Don't bother the user.

If **multiple branches** have unique commits (team-flow with long-lived `main` + `production` + `develop`, or active feature branches that haven't merged), fire a dedicated `AskUserQuestion`:

```
Question: "Your work touches multiple branches — which to include?"
Options:
- All branches, all refs (default — includes experiments, feature branches, tags)
- Main branch only
- "What shipped" only (main + production / release branches)
- Custom (I'll describe which to include or exclude in chat)
```

Whatever they pick, store in `state.scope.branch_filter` (default: `"--all"`; main-only: `"--branches=main"`; shipped-only: `"--branches=main --branches=production"`; custom: a list of `--branches` flags).

### 0c-ter — AI-framing (conditional, fires when there's a signal)

Run this once over the corpus:

```bash
git -C <repo> log <branch_filter> --no-merges --author='<author-regex>' --format='%B' \
  | grep -ciE 'Co-Authored-By:.*(claude|copilot|cursor|gpt|gemini|aider|codeium)'
```

Compare against the total commit count. **If less than 15% of commits have an AI co-author trailer**, skip this section — the user is writing mostly solo, no framing question needed. **Otherwise** fire one `AskUserQuestion`:

```
Question: "Most commits here have AI co-author trailers (Claude / Cursor / etc.).
           How do you want AI involvement framed across the book?"

Options:
- Don't mention AI involvement — write chapters as if solo work (Recommended for senior portfolios that want to stay focused on judgment, not tooling)
- One-line acknowledgement in the README header — chapters read as your work, but a line at the top of the book says "Chapters drafted in collaboration with [tool(s)]. Decisions, framing, and reflection are mine."
- Mention in every chapter — explicit AI acknowledgement woven into chapter prose where the work was clearly tool-driven
- Mention only where it shaped the work — skill weaves it in for chapters where commit signals show the work was code-generation-heavy; omits where signals suggest deeper human iteration (lots of small commits, reverts, fixes)
```

Default surfaced as "(Recommended)": **one-line acknowledgement in the README header**. It's honest without becoming the story.

Store the choice in `state.scope.ai_framing` (one of `none | readme-header | every-chapter | signal-based`). Phase 3 drafting and Phase 5 README generation read it.

**Why this lives at kickoff, not per-chapter:** for the target audience (people using AI coding tools daily), `Co-Authored-By: <tool>` is the rule, not the signal. Asking about every chapter would be 50 noise-questions. Asking once and applying uniformly respects the user's time and stays honest.

### 0d.1 — Format (one AskUserQuestion call, fires first)

The skill produces different artifacts depending on what the user is making. Ask this BEFORE the rest of the kickoff because format drives unit type, length, voice, interview depth, assembly, and visual companion path.

```
Question: "What are you making from these commits?"
Options:
- Long-form logbook / book (Recommended for portfolio depth — chaptered narrative, 30-60 chapters)
- Social post or carousel (LinkedIn, Instagram, X, Mastodon — platform asked next)
- Conference talk script (10-15 beats for spoken delivery, story arc)
- Personal journal / private log (loose dated entries, low-stakes, just for you)
```

Plus the auto "Other" → user describes the artifact (CV addendum, blog series, internal docs, etc.) and the skill adapts. Store in `state.scope.format` (one of `book | social | talk | journal | custom`).

### 0d.1-bis — Social platform (conditional, fires only if format == social)

```
Question: "Which platform are you posting to?"
Options:
- LinkedIn (carousel or single post — 1300-char post limit, hook-led, 200-600 words)
- Instagram (carousel + caption — image-led, 2200-char caption but readers skim the first 125)
- Twitter / X (thread — 280 chars per tweet, threaded for length)
- Multiple platforms (write once, the skill adapts copy per platform on export)
```

Plus auto "Other" for Mastodon / TikTok / Threads / etc. — user describes and the skill adapts character limits and conventions. Store in `state.scope.social_platform`.

See the `## Formats` section near the bottom of this file for the per-format and per-platform rules that Phases 2, 3, and 5 read.

### 0d.2 — Remaining kickoff questions (one AskUserQuestion call, max 4 items)

State the output dir as a default (don't ask, just say where it'll go). Then one batched AskUserQuestion with these:

1. **Granularity** — interpretation depends on format. For book: chapter count (default scaled to commit count — <100 → 10-20; 100-500 → 30-50; 500+ → 50+). For LinkedIn: post count (1 / 3 / 5-8). For talk: beat count (10-15). For journal: not asked, replaced with "what cadence — daily / weekly / per-commit?".
2. **Reflection layer** — interview style:
   - **Interviews, per-unit** — 3-5 questions after each draft, weave answers in before moving on. Highest quality, slowest. (For book / talk / linkedin where reflection adds value.)
   - **Interviews, batched** — draft several then loop through interviews. Compromise.
   - **Interviews, single end-pass** — draft everything first, run all interviews at the end. Fewest interruptions.
   - **No reflection** — technical-only output. No interviews. (Common for talks, CV addenda; rare for journal which is inherently reflective without prompts.)

   Default: per-unit. For journal format, default to "no formal interviews — user just writes whatever they want in the unit's reflection slot."
3. **Voice & tone source** — auto-detect first by looking for `CLAUDE.md`, `STYLE.md`, `TONE.md`, `VOICE.md`, `brand.md` in the repo. If found, surface the path and ask "use these + add anything?". If not found, ask for tone rules (banned words, brand voice, formal vs casual).

Time window is **not** asked here — default to all. If the user wants to narrow, they say so at the Phase 1 report step.

For **update mode**: skip both kickoff calls. Show "I see N new commits since [date]. Inheriting format, granularity, reflection, voice, branch scope from state. Confirm or override anything?" Only re-ask what looks wrong.

---

## Phase 1 — Corpus dump

1. Create output dir if missing. Sluggify the repo basename: lowercase, replace non-`[a-z0-9-]` with `-`, collapse repeats, trim. `My_Cool_Repo` → `my-cool-repo` → `~/my-cool-repo-logbook/`.

2. Build the author filter regex from confirmed identity (user emails + AI-tool bots the user confirmed driving). Alternation with `\|`.

3. Dump the filtered, non-merge log using the branch filter chosen in 0c-bis (default `--all` if the user wasn't asked):

   ```bash
   git -C <repo> log <branch_filter> --no-merges --reverse \
     --author='<author-regex>' \
     --format='COMMIT %h%nAUTHOR %an <%ae>%nDATE %ad%nSUBJECT %s%n%n%b%n=====%n' \
     > /tmp/<repo-slug>-commits-raw.md
   ```

4. **Don't fetch `--stat` for everything up front.** Per-chapter, lazily. When you do fetch `git show --stat <hash>`, cap visible files at 20 with `(...and N more files changed)` appended.

5. Report back briefly:

   ```
   Corpus: 587 in-scope commits, 2026-04-10 → 2026-05-20.
   Branch scope: all (470 from main, 12 from production, 8 from feat/old-experiment).
   Top themes by scope tag: dashboard 59 | companies 21 | chat 21 | landing 14 | …

   Moving to outline. Push back if anything looks off.
   ```

   **Squash-merge note:** If the repo uses squash-merge, you may see the original feature-branch commits AND the squash on main as separate items. Clustering in Phase 2 will surface this — flag it in the outline so the user picks the better representation per unit.

---

## Phase 1.5 — Score commits, cluster into narrative arcs (universal across formats)

This phase runs for **every format**. The output is a list of narrative arcs — coherent stories built from clusters of related commits, each ready to become a unit (chapter / post / beat / entry) in Phase 2.

Two passes:

1. **Score commits** for fit using the heuristics below.
2. **Cluster the top-scoring commits into arcs** using the arc-clustering signals further down.

The arc, not the commit, is the unit of writing. This is the difference between a changelog read aloud and something a stranger wants to read.

**Scoring heuristics** — each commit gets a fit score:

| Signal | Weight |
| --- | --- |
| Commit body length > 200 chars (has reasoning, not just a subject) | +3 |
| Subject contains impact verbs ("delete," "remove," "rewrite," "ship," "kill," "drop," "fix," "speed up," "rebuild") | +2 |
| Subject contains numeric impact ("from X to Y," "Nms → Mms," "Nx faster," "−N lines") | +3 |
| Files changed > 5 (significant scope) | +1 |
| Files changed > 50 (huge structural shift) | +3 |
| Part of a multi-commit arc (3+ related commits in a 7-day window with overlapping scope tag) | +2 |
| Subject is "wip:", "fix typo", "chore:", "ci:", "style:", "format", "bump deps" | −5 |
| Diff is < 5 lines net (trivial polish) | −2 |
| No commit body at all | −1 |

**Universal rule: every unit is a narrative arc, never a single commit.** Commits are raw material; the unit is the *story* you tell with them. A chapter, a post, a beat, an entry — each is built from a cluster of commits that share a tension, a setup, and a payoff. Writing commit-per-unit produces a changelog read aloud; writing arc-per-unit produces something a stranger wants to read.

Rank commits by score, then cluster the top commits into **narrative arcs** using these heuristics:

| Signal that commits belong in the same arc | Why |
| --- | --- |
| Overlapping scope tag (`feat(chat)`, `fix(chat)`, `refactor(chat)`) within 14 days | Same problem being worked on |
| Same touched files across multiple commits | Same code surface |
| A `feat` followed by ≥2 `fix` commits with related subjects | The "shipping it then iterating" arc |
| Commits referencing each other ("revert of X", "follow-up to Y") | Explicit chain |
| A `revert:` commit pointing back to a recent commit | The honest "we tried X, walked it back" arc — keep this, it's good narrative texture |
| Commits in the same time window mentioning the same concept (shared nouns in subjects/bodies) | Same conceptual thread |
| Distant commits with completely unrelated scope tags | Separate arcs — don't merge for tidiness |

Each arc gets:
- **Anchor** — the highest-scoring commit in the cluster (becomes the lead in the writing)
- **Supporting commits** — 2-12 surrounding commits that contribute detail
- **Arc title** — names the *story*, not the area. Good: "The week we deleted the matching algorithm." Bad: "Matcher refactor."
- **Tension** — what was the central problem / decision / surprise
- **Payoff** — what changed, what shipped, what was learned

**Arc-shortlist count per format:**
- **`book`:** typically 30-60 arcs (full corpus expressed as arcs; some arcs may be small, that's fine — a tight 200-word chapter is allowed)
- **`social`, single post:** 1 arc (the strongest story)
- **`social`, carousel (LinkedIn / Instagram):** 3-8 arcs (each its own post or carousel-slide-cluster)
- **`social`, thread (Twitter):** 1-3 arcs (each becomes a thread)
- **`talk`:** 3-5 arcs (each becomes a beat in the overall talk arc)
- **`journal`:** depends on cadence — per-commit-cluster (one arc per period) OR per-commit (no clustering). User picks at kickoff.
- **`custom`:** ask the user how many arcs

Present the arc shortlist (or arc outline for book) with anchor + tension + payoff so the user judges story-fit, not commit importance:

```
Arc shortlist for your LinkedIn carousel — 5 stories I'd build posts around:

1. "The week we deleted the matching algorithm" (2026-04-22 → 2026-04-26)
   Anchor: f2a18e9 chore: replace matcher with three SQL filters
   Supporting: c81da7e, b2117e3, 4a991ee
   Tension: 600-line scoring system we'd built carefully; couldn't tell scores apart
   Payoff: 12 lines of SQL, 800ms → 12ms latency, hardest deletion of the quarter

2. "When the chat input wouldn't stay visible" (2026-04-15 → 2026-04-17)
   Anchor: a91c4f2 fix: keep text input visible alongside chips
   Supporting: e6c1f0a, b22ad11
   Tension: chip-only UI felt cleaner but couldn't handle anything unanticipated
   Payoff: input always visible, chips as scaffolding — foundational decision

3. ...

Want to swap arcs, merge/split any, or cut ones that don't belong?
Otherwise I'll move to outline.
```

Wait for trim/swap/approve. Save the arc shortlist to `<output-dir>/.shortlist.md`. Then Phase 2.

**For book format specifically:** the arc list IS the chapter outline. Phase 2 just confirms / reorders. For other formats, Phase 2 takes the approved arcs and decides ordering / structure within the format.

---

## Phase 2 — Outline (hard gate)

Phase 1.5 produced the arc list. Phase 2 turns that into a final outline for the chosen format. The "unit" depends on `state.scope.format`: chapter for book, post for social, beat for talk, entry for journal. See `## Formats` for per-format unit rules.

1. **Map arcs → units.** Each arc from Phase 1.5 becomes one unit. For book, this may include small arcs (a tight 200-word chapter is fine). For social/talk, the user already trimmed in Phase 1.5; just confirm the mapping. For journal with per-commit cadence, units don't cluster — one entry per commit.

2. **Order arcs intentionally.** Default chronological for book and journal. For social, order by impact (the strongest hook first if posting in sequence). For talk, order for narrative arc (setup → escalation → climax → payoff across beats).

3. For each unit emit:
   - **Working title** in sentence case. The story, not the area. Good: "The referral loop that doesn't trigger on clicks." Bad: "Referral feature."
   - Date range
   - Anchor commit + supporting commits (from Phase 1.5)
   - **Tension** (one sentence — the central problem / decision)
   - **Payoff** (one sentence — what changed)

4. Save the outline to `<output-dir>/.outline.md`. Show the user the full list.

5. **Wait for green light.** Splits / merges / renames / reorders happen here, before prose. Cheap now, expensive later.

---

## Phase 2.5 — Calibration draft (write one, confirm direction, then continue)

After the outline is approved, write a draft of the FIRST unit only — chapter 1, post 1, beat 1, or entry 1 depending on format. Just the technical narrative; no reflection / interview yet. Save it to its final path.

Then ask the user:

```
Question: "Calibration draft saved as <path>. Before I burst through the rest, is this the right direction?"

Options:
- Yes, keep going — voice, depth, and angle feel right
- Voice is off — too AI-flavoured / wrong register (you'll tell me what to change)
- Angle is off — I'd frame this differently
- Depth is off — too long / too short / wrong level of detail
```

If "yes": proceed to Phase 3 for remaining units. No more checkpoints.

If anything else: discuss with the user, recalibrate the voice rules / shortlist / outline as needed, redo the calibration draft, ask again. Don't proceed until the user signs off.

**Skip Phase 2.5 entirely for `journal` format** — journal entries are low-stakes and self-directed; a calibration check is overkill. For all other formats, this is the most valuable single phase: it costs one unit of work and saves the cost of redoing 50 if the voice was off.

---

## Phase 3+4 — Per-unit draft + reflection (driven by the reflection setting)

For each approved unit after the calibration draft, in outline order. If `scope.reflection == "none"`, **skip Phase 4 entirely** — write technical-only units, no question blocks, no "From me" sections, and note in the output README header that this is a technical-only output.

### Draft (Phase 3)

1. Re-read that chapter's commits from the raw dump.
2. If the diff alone doesn't carry intent (vibe-coded chapters, cryptic messages, complex refactors), spawn an `Explore` agent on relevant code to map what changed. Keep lean — only when needed.
3. Draft each unit as a **narrative arc, not a commit summary**. Use the arc's anchor + supporting commits as raw material; tell the *story*. Apply the **AI-framing rule** from `state.scope.ai_framing` (set during kickoff, see 0c-ter) uniformly across the unit.

**Storytelling structure (applies to every format):**

Every unit follows a five-beat arc, adjusted in length and emphasis per format:

1. **Hook** — first sentence drops the reader into the moment. Specific, concrete, dropped *in medias res*. Not the area of work, the *moment*.
2. **Setup** — what state was the system / decision / mind in before. Short.
3. **Tension** — what was the problem, the doubt, the wrong assumption, the thing that broke. The honest emotional beat.
4. **Specifics** — what was tried, what got cut, what shipped. Cite commits inline as `\`a91c4f2 fix: keep text input visible alongside chips\``. Specific verbs, concrete nouns, real numbers.
5. **Payoff** — what changed, what was unlocked, what was learned. One line. Land the plane.

Format-specific dial settings:
- **book / journal:** all five beats over 3-8 paragraphs (book) or 50-500 words (journal). Reflective pacing.
- **social:** all five beats compressed into 200-600 words for a single post; for a carousel the beats can spread across slides (slide 1 hook, slides 2-4 setup+tension+specifics, slide 5 payoff). Hooks must work in the first line — LinkedIn truncates after ~2 lines in feed view.
- **talk:** beats become the talk arc itself. A 10-15 beat talk has setup beats, tension beats, specifics beats, payoff beats — the whole thing reads as a story, not a tour.

**Voice rules — strict (apply to every format):**

- Start in medias res. First sentence shows the user *in* the work. Forbidden openers: "In this chapter," "Let's look at," "This chapter is about," "We'll explore," "It's important to note that."
- **Forbidden phrases anywhere in unit prose:** "amazing," "innovative," "passionate," "game-changer," "unlock potential," "cutting-edge," "world-class," "let's dive into," "it's worth noting," "important to remember," "it's clear that," "needless to say." Also no em dashes in prose (en-dash, period, comma, colon, parens are fine).
- No closing recap. When the point is made, stop. Don't end with "and that's how chapter N taught me Y."
- No uppercase tracking-wide kickers / eyebrows.
- Sentence case for headings.
- Honest about slow parts. "The fix took twenty minutes; the decision took two days" is the energy.
- **Specifics over abstractions.** "12 lines of SQL" beats "a refactor." "800ms → 12ms" beats "much faster." "Two days defending the chip-only flow" beats "took a while to accept." If you can't make it specific, cut it.

**Social-format extra rules (apply when format == social):**

- Hook in the first line — feed view truncates around 200 chars on LinkedIn, 125 on Instagram, full 280 on Twitter. Get to the moment before the truncate.
- No "X lessons from Y" framing. No "I used to think X, then I realized Y." No manufactured-tension hooks ("WAIT.", "Most people get this wrong.", "Here's what nobody tells you about Z.").
- No engagement bait. No "Agree? Drop a 👍," "Comment INSIGHTS for the deck," "↓ what would you add?", "Repost if this helped." If the writing is good, the engagement comes — begging for it is the tell.
- No hashtag spam. LinkedIn / Instagram / X conventions differ (LinkedIn 3-5 sometimes, Twitter zero, Instagram more) but skill default is **zero hashtags** unless user asks.
- No emoji decoration of headlines / hooks. Emojis inside body text are fine when they carry meaning, but using one as a bullet or a vibe-setter at line start reads as performative.
- One insight per post. If two ideas want in, that's two posts.
- The post ends on the payoff, not a call to action. Trust the reader.

**Don't fabricate.** If a commit's purpose isn't clear from message + diff, do **not** invent reasoning. Either: (a) describe what changed at the file level without claiming intent, OR (b) if reflection is enabled, flag it in the "questions I need answered" block at the bottom; if reflection is disabled, leave a `> TODO: clarify intent here` marker that the user can fill in or delete.

**If reflection is enabled**, end the draft with:

```
### From me — questions I need answered

1. ...
2. ...
```

Three to five questions. Quality bar:

| Good | Bad |
| --- | --- |
| "What was the moment you realised the chip-only flow was wrong?" | "Why did you do this?" (too open) |
| "Did you push back on the chip approach, or ship it first?" | "What did you learn?" (yields generic) |
| "Was the input-always-visible idea yours, your collaborator's, or did it emerge from a conversation?" | "Was this hard?" (yes/no) |
| "What was the first version you scrapped, and why?" | "How do you feel about the result?" (sentimental) |

### Interview (Phase 4)

**Skip this section entirely if `scope.reflection == "none"`.**

Otherwise, the timing comes from `scope.reflection`:

- **`per-unit`:** ask immediately after drafting via AskUserQuestion (4 per call; if you wrote 5 questions, use two calls — first 3, then the remaining 2).
- **`batched`:** draft 5-10 units then loop through interviews.
- **`end-pass`:** draft all units with question blocks, do all interviews at the end.

### Weaving the answers — strict rules

The reflection section is the **user's voice**. Light syntax tidy only. Never paraphrase, expand, or "elevate."

Examples of correct weaving:

| User typed | Section reads |
| --- | --- |
| `yeah it was the wsl thing i didnt realise paths broke on save` | "Yeah, it was the WSL thing. I didn't realise paths broke on save." |
| `ruben pushed for chips i pushed back later, was wrong at first` | "Ruben pushed for chips. I pushed back later. Was wrong at first." |
| `lol i didnt even know what active storage was` | "I didn't even know what Active Storage was." |

What NEVER to do:

| User typed | DO NOT write |
| --- | --- |
| `wsl thing, paths broke on save` | "It was a fascinating discovery about WSL's path-handling behaviour, which revealed how subtle environment configurations can shape a developer's workflow." |

**If the user doesn't answer a question:** leave the section blank with `> TODO: <question>` instead. Never fabricate.

### Save

Each unit saves to its format-specific file path (see `## Formats`):
- `book` → `chapter-NN-<slug>.md`
- `social` → `post-NN-<slug>.md` (or `carousel-<slug>.md` / `thread-<slug>.md`)
- `talk` → `beat-NN-<slug>.md`
- `journal` → `entry-YYYY-MM-DD-<slug>.md`
- `custom` → user-defined pattern

Slug rule (all formats): lowercase, kebab-case, max 40 chars, drop punctuation, keep only `[a-z0-9-]`. Example: `chapter-14-referral-loop-doesnt-trigger-on-clicks.md`.

Update `<output-dir>/README.md` continuously so the index reflects current state. Useful if interrupted.

### Hand-off messaging after every save

Always remind the user, swapping unit name for format:

> Saved through [chapter / post / beat / entry] NN. You can stop anytime. Run the skill again later and it'll pick up from NN+1.

### State file after every save

Write `<output-dir>/.logbook-state.json`. Annotated example:

```jsonc
{
  "version": 1,
  "created_at": "2026-05-21T09:00:00Z",
  "last_updated": "2026-05-21T17:30:00Z",
  "repo_path": "/home/user/code/myrepo",
  "repo_slug": "myrepo",
  "last_commit_hash": "b7de93c",   // HEAD at time of last save — used for update detection
  "last_commit_date": "2026-05-20",
  "identity": {
    "user_emails": ["jane@example.com", "12345+jane@users.noreply.github.com"],
    "user_names": ["Jane Smith"],
    "ai_tools_as_user": ["v0[bot]"],
    "humans_acknowledged": [{"name": "John Doe", "emails": ["john@example.com"]}],
    "excluded": ["dependabot[bot]", "vercel[bot]"]
  },
  "scope": {
    "format": "book",                     // book | social | talk | journal | custom (set in 0d.1)
    "social_platform": null,              // linkedin | instagram | twitter | multiple | other — only set if format == social
    "granularity": "deep",
    "reflection": "per-unit",             // per-unit | batched | end-pass | none
    "time_window": "all",
    "branch_filter": "--all",             // raw git flag(s) from 0c-bis choice
    "ai_framing": "readme-header",        // none | readme-header | every-unit | signal-based (set in 0c-ter)
    "ai_tools_detected": ["claude", "cursor"],  // which AI tools appear in Co-Authored-By trailers
    "visuals": "none",                    // none | prompts | skill:<name> — set when social/talk visual companion runs
    "visual_style": null,                 // user-described default style if visuals == prompts
    "brand_style": null,                  // path to brand file OR inline brand JSON (palette/typography/mood/illustration_style/references) — set after the brand-style question fires
    "output_dir": "/home/user/myrepo-studio"
  },
  "voice_rules": "see ./CLAUDE.md, plus: no em dashes, no eyebrows",
  "chapters": [
    {
      "number": 14,
      "slug": "chat-input-that-wouldnt-stay-visible",
      "title": "The chat input that wouldn't stay visible",
      "file": "chapter-14-chat-input-that-wouldnt-stay-visible.md",
      "commits": [
        // Each commit stored as hash + subject + date + author_email.
        // This 4-tuple is what survives a rebase: if hash is gone,
        // we re-attach by matching (subject + date + author_email).
        {"hash": "a91c4f2", "subject": "fix: keep text input visible alongside chips", "date": "2026-04-22", "author_email": "jane@example.com"}
      ],
      "reflection_complete": true
    }
  ]
}
```

---

## Phase 5 — Highlights + final index

Once all chapters are written and reflections woven:

1. **`<output-dir>/README.md`** — table of contents, one-line abstract per chapter, total commits referenced, date span, identity bundle credited. If `state.scope.ai_framing == "readme-header"`, include a single line near the top of the README naming the tools detected, e.g.: *"Chapters drafted in collaboration with [Claude, Cursor]. Decisions, framing, and reflection are mine."* Pull tool names from `state.scope.ai_tools_detected`. If `ai_framing == "none"`, omit. If `every-chapter` or `signal-based`, mention in the chapters but skip the README header.

2. **`<output-dir>/highlights.md`** — 8-12 standout commits worth pointing at. One paragraph each on **significance**.

   **Highlights are about craft, not outcomes.** Write about why a decision is elegant, what it unlocked technically, what it taught. Do **not** write business-impact lines ("this drove 30% engagement," "users loved this") — those are unverifiable and read as LinkedIn cosplay. If the user wants impact metrics in their highlights, they can add them.

Quality check before declaring done: re-read README + three random units. Run `grep -c '—\|amazing\|innovative\|passionate\|cutting-edge' <output-dir>/*.md` — should be zero in prose. If anything is nonzero, fix it before signing off.

---

## Formats

Per-format rules that Phases 2, 3, and 5 read. One format picked at kickoff (`state.scope.format`), applied uniformly across the run.

### `book` — Long-form logbook

- **Unit:** chapter | **count:** 30-60 (driven by granularity, scaled to commit count)
- **Length:** 3-8 paragraphs per chapter, ~300-700 words
- **Voice:** the chapter-excerpt calibration target above. Reflective, paced, honest. Problem → approach → result → reflection.
- **Interview:** 3-5 questions per chapter
- **File:** `chapter-NN-<slug>.md`
- **Assembly:** README with TOC + abstracts + AI-framing line (if applicable), `highlights.md` with 8-12 standout commits

### `social` — Posts and carousels (umbrella; platform sub-rules below)

- **Unit:** post (or carousel = sequence of slide-posts) | **count:** depends on platform
- **Voice:** hook-led, one insight per post. **Forbidden tropes:** "X taught me Y" recaps, humble-brags, rocket emojis, "agree?" closers, "↓ in comments" CTAs that beg for engagement, "I used to think X, then I realised Y" frames. The post stands on the specifics, not the moral.
- **Interview:** 1-2 punchy questions per post (or per carousel as a whole)
- **File:** `post-NN-<slug>.md` per post, `carousel-<slug>.md` for multi-slide
- **Assembly:** posts ordered by suggested posting sequence in a `posting-order.md`, plus a `caption-pack.md` for multi-platform export
- **Visual companion:** see section below

#### Social platforms

| Platform (`social_platform`) | Hard limit | Carousel | Notes |
| --- | --- | --- | --- |
| **`linkedin`** | 1300 chars per post (gets truncated past) | 5-10 slides, ~80 words supporting copy + cover hook + closing slide | Hook on line 1, "see more" break by line 3 |
| **`instagram`** | 2200 chars caption (readers skim first 125 — front-load) | 10 slides max, image-led | 5-10 relevant hashtags appended |
| **`twitter`** | 280 chars per tweet | Thread: 5-12 numbered tweets, hook in #1, payoff in last | No hashtags (current convention) |
| **`multiple`** / **`other`** | Adapt to user-described conventions | — | Skill exports per-platform variants |

### `talk` — Conference / meetup script

- **Unit:** beat | **count:** 10-15 beats for a 20-30 minute talk
- **Length:** 50-150 words per beat (spoken cadence — read aloud during draft)
- **Voice:** spoken cadence, callbacks, story arc (setup → conflict → resolution → payoff). Sentences shorter than written prose.
- **Interview:** 3-5 story-shaping questions TOTAL (not per beat) — done before drafting beats
- **File:** `beat-NN-<slug>.md`
- **Assembly:** `script.md` with all beats in order + slide-title suggestions per beat, optional `speaker-notes.md`
- **Visual companion:** see section below

### `journal` — Personal log

- **Unit:** entry | **count:** one per commit cluster OR per day (user picks)
- **Length:** 50-500 words, can ramble
- **Voice:** low-stakes, audience-free, can be sloppy. The opposite of polished portfolio prose.
- **Interview:** none by default. The reflection slot is freeform — user writes whatever, no prompted questions.
- **File:** `entry-YYYY-MM-DD-<slug>.md`
- **Assembly:** flat index by date, no highlights doc
- **No calibration draft** (Phase 2.5 is skipped — journal is self-directed)

### `custom` — Other

User described the artifact (CV addendum, blog series, internal docs, README chapter for an open-source project, etc.). The skill asks follow-up questions in Phase 0d.2 or at the start of Phase 1.5 to fill in:
- Unit name (post / chapter / entry / section / something else)
- Count target
- Length per unit
- Voice & tone
- Interview style
- File path pattern
- Assembly structure

Then proceeds with custom rules.

---

## Visual companion (social and talk formats)

For `social` and `talk` formats, text alone isn't the artifact — posts need cover images, talk slides need visuals. Once per session, before Phase 3 drafting begins, the skill **detects what the current runtime can actually do** and offers options dynamically.

### Detection (runs silently before asking)

Probe in this order and remember what's available:

1. **Native runtime image generation** — does the agent runtime have a built-in image-generation tool?
   - Codex / OpenAI CLI: `openai images.generate` or runtime image tool
   - Gemini CLI: `gemini image` or runtime image tool
   - Claude Code with image MCP server connected (look for MCP tools matching `*image*`, `*generate-image*`, etc.)
   - Any agent runtime that exposes an image-gen tool through its normal tool surface
   - Probe by checking available tools / `which <cli>` for known image CLIs

2. **Installed image-gen skills** — scan `~/.claude/skills/` and similar (the agent-specific paths from the agent-skills CLI table) for skills whose `name` or `description` suggests image generation: `gpt-image`, `nano-banana`, `image-gen`, `sora`, `flux`, `midjourney-*`, `brandkit` (for brand boards), `imagegen-frontend-web`, `imagegen-frontend-mobile`, `lottie-animator` (for animation).

3. **Installed design / mockup skills** — scan for skills that produce designed artifacts you could screenshot or render: `frontend-design`, `figma` MCP, `canva` MCP, `industrial-brutalist-ui`, `minimalist-ui`, `high-end-visual-design`. These design slides/posts as HTML/CSS or in Figma/Canva — different output mode than pixel image gen.

4. **CLI fallbacks** — bash-invokable image tools the user has installed locally: `gemini`, `openai`, custom scripts in `~/bin/`, etc.

### Ask once, with options shaped by what was detected

Build the AskUserQuestion options dynamically. Always include "give me prompts" and "no" as fallbacks. Bubble direct-generation and skill hand-off to the top when available. Mark the strongest available option (Recommended).

**Example A — runtime has native image gen + frontend-design installed:**

```
Question: "Visuals for the [posts / slides]?"
Options:
- Yes, generate images directly with [<detected runtime image tool>] (Recommended — produces real image files in your output dir)
- Yes, design them as code with frontend-design (Vue/HTML mockups, renderable to PNG via screenshot)
- Yes, give me paste-ready prompts for external tools (ChatGPT / Nano Banana / Midjourney / Gemini Image)
- No, text only
```

**Example B — no native image gen, gpt-image skill installed, frontend-design installed:**

```
Question: "Visuals for the [posts / slides]?"
Options:
- Yes, generate images via the gpt-image skill (Recommended)
- Yes, design them as code with frontend-design
- Yes, give me paste-ready prompts for external tools
- No, text only
```

**Example C — nothing detected:**

```
Question: "Visuals for the [posts / slides]?"
Options:
- Yes, give me paste-ready prompts (you'll paste into ChatGPT / Nano Banana / Midjourney / Gemini)
- No, text only
```

Store the choice in `state.scope.visuals` as one of:
- `runtime:<tool-name>` — direct generation via runtime tool
- `skill:<skill-name>` — hand-off to an installed skill
- `prompts` — paste-ready prompts in unit files
- `none` — text only

### Behaviour per choice

**`runtime:<tool>` (direct generation):**
For each unit, build the same kind of prompt as for the "prompts" option (subject, mood, style, aspect, brand constraints), then INVOKE the runtime image tool with that prompt and save the resulting image to `<output-dir>/images/unit-NN-<slug>.png`. Reference the image path in the unit's markdown file rather than appending a prompt block. Don't ask the user to paste anything.

**`skill:<skill>` (hand-off):**
For each unit, hand off context to the chosen skill — the unit's text + a one-sentence visual direction + the brand JSON (if resolved in Phase 0 brand-style). The hand-off skill produces its native artifact (Figma file, Canva design, HTML/CSS component, etc.). The skill captures the output reference and stores it alongside the unit. Don't re-ask which skill per unit.

**`prompts` (paste-ready):**
For each unit, append a `### Image prompt` block to the unit's file. Each prompt specifies:
- **Subject and mood** (derived from the unit's core idea / arc tension)
- **Style hint** (from the brand archetype, or ask once for default style preference)
- **Aspect ratio** (1:1 for LinkedIn carousel slides; 4:5 for Instagram; 9:16 for stories; 16:9 for talk slides)
- **Constraints** (no text in image unless intentional; respect brand colors from `state.scope.brand_style`)
- **Where to paste** — name the most likely tool ("Paste into Nano Banana, ChatGPT image, Midjourney v7, or Gemini Image")

Example block appended to a LinkedIn carousel post:

```
### Image prompt

Aspect ratio: 1:1 (LinkedIn carousel slide)
Brand palette: ink #0B132B, clay #D64A1A, cream #F4F1E8
Style: clean editorial illustration, muted palette

A share button frozen mid-press. No confetti, no animation, no
celebration — just the moment of intent. Background empty. The
button looks ready but the system is silent. Minimal, deliberate.

(Paste into Nano Banana, ChatGPT image, Midjourney v7, or Gemini Image.)
```

**`none`:** skip the visual companion. Pure text output.

**Don't ask per unit.** One choice at the start of Phase 3, applied across all units in the run.

**Mixed mode (advanced):** if the user explicitly says "use the runtime for hero images but give me prompts for the rest" — fine, honour that. Store as `runtime+prompts` and produce both per unit.

### Brand style (conditional, fires only when visuals are requested)

Visual consistency across posts / slides is the difference between "this looks designer-made" and "this looks AI-generated." Before the first prompt or skill hand-off, ask once:

```
Question: "Do you have a brand style to apply to the visuals?"

Options:
- Yes, pull from this codebase (Recommended if there's a brand guide here — I'll look for `brand.md`, `STYLE.md`, `DESIGN.md`, `tailwind.config.*`, CSS custom-property tokens in `application.css` / `globals.css`, or a tokens / theme JSON file)
- Yes, from a separate file — paste the path
- No, help me create one — I'll ask brand questions (palette, type, mood) and use a taste skill if one is installed (brandkit, stitch-design-taste, high-end-visual-design) to compile a coherent brand
- Choose for me — I'll propose a brand based on the content's tone and format, show it for one-tap confirm, then apply
```

Resolve the choice and store in `state.scope.brand_style` as one of:
- A path to a brand file found / provided
- Inline JSON with the resolved brand:
  ```json
  {
    "palette": [{"name": "ink", "hex": "#0B132B"}, {"name": "clay", "hex": "#D64A1A"}, {"name": "cream", "hex": "#F4F1E8"}],
    "typography": {"display": "Söhne Breit / similar", "body": "Inter / similar"},
    "mood": ["grounded", "specific", "no hype"],
    "illustration_style": "clean editorial, muted palette, no gradients",
    "references": ["https://...", "internal: /docs/brand"]
  }
  ```

**Auto-detection rules (option 1, "pull from codebase"):**
- Scan repo root for files matching `brand.md|STYLE.md|TONE.md|VOICE.md|DESIGN.md|brand-guidelines.md`
- Scan `tailwind.config.{js,ts,cjs,mjs}` for theme.extend.colors / fontFamily
- Scan `*.css` files for `:root { --color-* }` custom properties
- Scan for `tokens.json` / `theme.json` / `design-tokens.json`
- If `CLAUDE.md` mentions brand colors / type / voice, extract
- Present what you found in one message ("I see clay orange #d64a1a, night blue #0f1a2a, warm cream #f4f1e8 in application.css plus a voice section in CLAUDE.md — use these?") and confirm

**Brand-creation flow (option 3, "help me create"):**
The skill has a built-in **brand taste library** (see below) with twelve named archetypes, per-archetype deep guidance (visual qualities, fonts, references, lean-into / avoid), categorised anti-AI-slop rules, type-pairing recipes, palette recipes, and a brand-board image-prompt generator. The library is the deep treatment, fully self-contained — no other skill needs to be installed.

Walk the user through:
- Three adjectives for the mood (or point them at the archetype map below)
- Palette: warm / cool / monochrome / high-contrast / muted
- Type: serif / sans / mixed / display-heavy / utilitarian
- One reference (an existing brand, site, or visual they want to feel like)

Match their answers against the closest archetype, apply its per-archetype guidance, and compile into the brand JSON.

**Optional external override:** if `~/.claude/skills/` or active MCP servers contain `brandkit`, `stitch-design-taste`, `high-end-visual-design`, `minimalist-ui`, or `industrial-brutalist-ui`, you may surface them once as "I have these installed — want me to override the built-in library with one of them instead?" If user picks one, hand off with the artifact's format + tone context. Otherwise default to the inline library.

**Choose-for-me flow (option 4):**
Look at the corpus content (tone of commits, scope tags, what the work is about — dev tool? consumer product? B2B SaaS? indie hacker project?), the format (social vs talk vs custom), match against the built-in archetype library, propose the best-fit archetype in one short paragraph + the resolved brand JSON. User confirms in one tap or says "different vibe — try X" and you propose another.

### Built-in brand taste library (deep, self-contained)

Twelve archetypes covering most portfolio-content vibes. The library is comprehensive enough on its own — installed taste skills are an entirely optional layer for users who want a different opinionated voice.

#### Archetype map

| Archetype | Palette feel | Typography | Mood | When it fits |
| --- | --- | --- | --- | --- |
| **editorial** | restrained neutrals + one accent | display serif + humanist sans body | reflective, paced, designed | long-form portfolio, deep writing, magazine-feel social |
| **tech-minimal** | monochrome + one bright accent | neue grotesk display + same-family body | clear, restrained, modern | developer tools, infra, B2B SaaS, dev-audience LinkedIn |
| **warm-indie** | earthy warm (clay, cream, ink) | display serif or rounded sans + warm body sans | grounded, human, specific | indie hacker, solo founder, "made by a real person" |
| **industrial-brutalist** | high-contrast B&W + one signal color | mono + utilitarian tight sans | direct, exposed, no-decoration | infra, devtools, declassified-document aesthetic |
| **premium-luxury** | deep saturated + ivory + metallic accent | refined classical serif + minimal sans body | considered, restrained, intentional | high-end consumer, agency work, polished portfolio |
| **dark-tech** | dark surfaces + neon/cyan accent + mono code | sans + monospace pairing | technical, late-night, hacker | terminal apps, security tools, dev-tool launches |
| **documentary** | muted desaturated + restrained accent | neutral sans + serif for emphasis | factual, restrained, observational | research notes, postmortems, technical writeups |
| **sharp-modern** | high-contrast color + neutral base | display sans with personality + clean sans body | confident, opinionated, current | consumer products with attitude, agency work |
| **cultural-zine** | bold flat color (red/black/cream) | display serif or expressive sans + workhorse body | expressive, opinionated, hand-made | manifesto-style content, cultural commentary, zine carousels |
| **soft-modern** | warm pastel + cream + ink | rounded display sans + same-family body | optimistic, friendly, modern | consumer apps, friendly indie products, onboarding content |
| **maker-craft** | natural / earthy + warm wood / paper textures | humanist serif + characterful sans | crafted, lived-in, slow | physical-product makers, craft tools, artisanal indie |
| **security-terminal** | near-black + amber/green terminal + bone | mono primary + tight sans | classified, technical, intentionally vintage | security tools, infra dashboards, "looks like a NOC" content |

For each archetype, the resolved brand JSON looks like:

```json
{
  "archetype": "warm-indie",
  "palette": [
    {"name": "clay", "hex": "#D64A1A", "role": "accent — sparingly"},
    {"name": "night", "hex": "#0F1A2A", "role": "text + dark surfaces"},
    {"name": "cream", "hex": "#F4F1E8", "role": "background — dominant"}
  ],
  "typography": {
    "display": "GT Sectra / Söhne Breit / similar display serif or strong sans",
    "body": "Inter / Söhne / readable humanist sans, 16-18px"
  },
  "mood": ["grounded", "specific", "no hype"],
  "illustration_style": "minimal editorial line work, muted palette, no gradients, lots of negative space",
  "references": ["Stripe Press", "Linear changelog", "Anthropic site"]
}
```

#### Per-archetype deep guidance

When an archetype is selected (by the user, by the corpus tone, or by "choose for me"), apply this guidance verbatim when writing the brand JSON and the image prompts:

**editorial** — Long-form publication aesthetic. Visual qualities: generous whitespace, baseline grid alignment, single accent colour used sparingly, photo treatment leans toward duotone or restrained colour. Fonts: GT Sectra, Tiempos, Söhne Breit, NYT Cheltenham for display; Söhne, Inter, Source Serif for body. Lean into: photo-led hero compositions, long captions, drop caps used sparingly. Avoid: serif body type with display sans (it inverts the convention).  
*References:* Stripe Press, Anthropic site, The Browser Company blog, NYT Cooking.

**tech-minimal** — The "Linear / Vercel / Stripe" aesthetic. Visual qualities: monochrome surface with one saturated accent, hairline borders, generous internal padding, charts that respect a 4 or 8px grid. Fonts: Inter, Söhne, Helvetica Now, GT America for display + body in same family at different weights. Lean into: type-led layouts, single-accent CTAs, dot-grid backgrounds at low opacity. Avoid: gradients, drop shadows beyond 8% opacity, "glassmorphism."  
*References:* Linear, Vercel, Stripe Docs, Resend.

**warm-indie** — Solo-founder / indie-hacker vibe. Visual qualities: cream or off-white background, warm dark text (#0F1A2A type), single warm accent (clay, terracotta, rust), illustration leans toward hand-drawn line work or simple shape compositions. Fonts: GT Sectra or Söhne Breit for display, Inter / Söhne for body, occasional mono. Lean into: hand-drawn touches, asymmetric photo crops, personal-feeling type. Avoid: corporate stock photography, gradients, slick over-rendering.  
*References:* Geoffrey Litt site, Maggie Appleton site, Robb Knight site, Stripe Press.

**industrial-brutalist** — Swiss print meets military terminal. Visual qualities: high contrast (near-black on bone), exposed grids, technical labels, classified-document feel, single signal accent (often orange, red, or amber). Fonts: IBM Plex Mono + IBM Plex Sans, GT America Mono, Inter Tight. Lean into: visible baseline grids, label noise ("FIG. 01-A"), data tables, technical numeric callouts. Avoid: anything that softens the rigor (rounded corners, gradients, illustration).  
*References:* Are.na, Read.cv, Field Notes packaging, declassified CIA documents.

**premium-luxury** — Considered, expensive, intentional. Visual qualities: deep saturated tones (oxblood, forest, midnight) on ivory or bone, refined serif at large sizes, generous negative space, photo treatment with film grain and considered color grading. Fonts: ABC Diatype Mono, GT Super Display, Garamond, Bembo for display; ABC Diatype, Söhne for body. Lean into: large display type, intentional cropping, considered photo direction. Avoid: trend-chasing colors, drop shadows, anything overworked.  
*References:* Aesop, Off-White editorial, Apollo Magazine, Atelier Olschinsky.

**dark-tech** — Late-night hacker aesthetic. Visual qualities: near-black surfaces (#0A0A0A not #000), single saturated accent (cyan, electric green, amber), monospace integrated naturally into the design, terminal-inspired layouts. Fonts: JetBrains Mono, IBM Plex Mono, Berkeley Mono + Inter Tight, Söhne Mono. Lean into: code samples as design elements, mono numerics, ASCII textures used sparingly. Avoid: clichéd "Matrix" green-on-black, neon glow effects, cyberpunk tropes.  
*References:* Linear's dark mode, GitHub dark, Fly.io, Vercel dark mode.

**documentary** — Research-paper aesthetic. Visual qualities: muted desaturated palette, serif emphasis only on specific words, neutral sans body, photo treatment B&W or muted color, lots of negative space. Fonts: Söhne, Inter, IBM Plex Sans for body; Tiempos or Source Serif for emphasis only. Lean into: footnotes, sidebar annotations, factual headlines. Avoid: emotional language in headlines, decorative flourishes, color.  
*References:* The Pudding articles, Bloomberg Originals, Pitchfork Sunday Review.

**sharp-modern** — Confident agency / contrarian-take aesthetic. Visual qualities: bold high-contrast color pairings (one neutral + two saturated), expressive display type, asymmetric layouts, large type set against negative space. Fonts: Migra, Söhne Breit, ABC Whyte for display; Söhne, Inter for body. Lean into: oversized headlines, intentional letter-spacing tweaks, single hero color taking up real estate. Avoid: trying too hard, pure trend-chasing, type tricks that distract from message.  
*References:* Pentagram client work, Order.design, MetaLab, Sandwich.co.

**cultural-zine** — Manifesto / poster aesthetic. Visual qualities: flat bold colors (often red + black + cream), expressive display type sometimes hand-drawn, screen-printed feel, intentional "imperfections," collage compositions. Fonts: hand-drawn or expressive display (custom or Migra, NaN Holo Demo, Söhne Breit Extended); plain sans body. Lean into: large word fragments as art, registration-misalignment as design, paper-texture backgrounds. Avoid: digital cleanliness, gradients, refined typography (it kills the energy).  
*References:* Are.na manifestos, Adbusters, Eye Magazine, Pentagram poster work.

**soft-modern** — Friendly consumer aesthetic without becoming twee. Visual qualities: warm pastel backgrounds (cream, soft peach, sage), rounded but not childish, illustration leans toward simple flat with a hand-drawn quality. Fonts: rounded sans display (ABC Whyte Round, Söhne Schmal, Söhne Mono Halbfett) + same-family body. Lean into: warm peach/cream backgrounds, single accent ink, illustrations that feel hand-finished. Avoid: emoji decoration, pastel-on-pastel low contrast, cartoony illustration.  
*References:* Anthropic site, Linear changelog illustrations, The Browser Company onboarding.

**maker-craft** — Physical-product / artisan feel. Visual qualities: natural textures (paper, wood grain), warm muted palette (taupe, sand, deep red, ink), serif type with character, photo treatment warm and slightly grainy. Fonts: GT Sectra, ABC Diatype Mono, Söhne with a serif partner. Lean into: object photography, hand-written notes as design elements, paper textures, considered material palette. Avoid: digital-only rendering, slick photography, anything that hides the hand.  
*References:* Field Notes, Best Made Co archives, Pre-loved tools project.

**security-terminal** — Specifically for security / infra tools. Visual qualities: near-black background, amber or green CRT-style accent, monospace dominant, fake "classified" label noise tasteful, no neon. Fonts: Berkeley Mono, IBM Plex Mono, JetBrains Mono. Lean into: terminal-inspired layouts, monospace heads, technical numeric callouts, redacted-rectangle as design element used sparingly. Avoid: hacker cliché (green-on-black Matrix glow), padlock icons, "cyber" anything.  
*References:* 1Password documentation, Fly.io docs, Tailscale brand.

#### Brand-board image prompt generator

Once the brand JSON is resolved, optionally generate a single image prompt that produces a "brand board" — a one-image summary of the brand the user can paste into ChatGPT / Nano Banana / Midjourney to get a visual reference for the whole identity. Ask:

```
Question: "Want a brand-board image prompt so you can see the brand as one image?"
Options:
- Yes, generate prompt
- No, skip — I have what I need
```

If yes, produce a prompt of the form:

```
### Brand board prompt

Aspect ratio: 16:9
Style: clean editorial brand-board layout, designer's working document

A brand identity board for [archetype name] presented as a single image.
Composition: top-left, three large color swatches labeled by name + hex
([palette colors from JSON]). Top-right, type specimens showing display
[display font] and body [body font] at large size — show a sample headline
and a paragraph of lorem ipsum. Mid-band, three small mood-image
references showing the visual mood: [archetype's visual qualities phrase].
Bottom band, three sample applications mocked up small: a post composition,
a slide composition, a small social tile — all in the brand palette using
the typography. Background of the brand board itself: bone/cream (#F8F5EE).
Whole thing reads like a designer's working brand reference document. No
text overlays; let the type specimens speak.

(Paste into Nano Banana, ChatGPT image, Midjourney v7, or Gemini Image.)
```

Save the brand board prompt to `<output-dir>/.brand-board-prompt.md` so the user can revisit.

#### Categorized anti-slop rules

**Color slop (skip these by default):**
- Purple-pink-blue gradients (the OpenAI / DALL-E default)
- Trend pairings that age fast (2020 lavender + sage; 2023 baby-blue + peach; 2024 lime + coral)
- Pure black (#000) or pure white (#FFF) for designer-feel surfaces; use `#0A0A0A` and `#FAFAFA` or warmer alternatives
- More than 5 hues in a single composition

**Layout slop:**
- Floating 3D rendered objects in negative space (Midjourney v5 default)
- Centered everything (the SaaS default)
- "Diverse team in modern office" stock-photo composition
- Card-inside-card-inside-card nesting
- Hero text on blurred photo background

**Typography slop:**
- Lato + Open Sans pairing (the SaaS default)
- Lobster, Pacifico, Bebas Neue (overused display fonts)
- All-caps body text
- Two serifs paired unless intentional (editorial archetype only)
- Letter-spacing tracked out beyond +50 in body copy

**Illustration / imagery slop:**
- Literal rendering of metaphors ("brain made of circuits," "lightbulb of ideas")
- Infographic-style thin-outlined icon clusters in pastels
- Glowing edges, lens flare, "energy" effects on UI
- Rocket emoji 🚀, "to the moon" anything
- Generic "futuristic cyberpunk neon" unless the archetype explicitly calls for it
- Stock-photo perfectly-diverse team huddled around a laptop

**Composition slop:**
- Symmetric centered layouts when asymmetric would have more energy
- Identical card sizes across a whole composition
- Right-text / left-image repeated for every section
- Photos with the same crop ratio used over and over

#### Type pairing recipes

Specific pairings that work, by archetype:

| Pairing | Best for archetype | Vibe |
| --- | --- | --- |
| Söhne Breit + Inter | tech-minimal / sharp-modern | confident, modern, restrained |
| GT Sectra + Söhne | editorial / warm-indie | reflective, designed, paced |
| Tiempos + Source Sans | documentary / editorial | factual, considered |
| ABC Diatype Mono + ABC Diatype | premium-luxury | intentional, expensive |
| IBM Plex Mono + IBM Plex Sans | industrial-brutalist / security-terminal | technical, exposed |
| Berkeley Mono + Inter Tight | dark-tech / security-terminal | hacker, late-night |
| Migra + Söhne | sharp-modern / cultural-zine | bold, opinionated |
| ABC Whyte Round + ABC Whyte | soft-modern | friendly, modern, not twee |
| Garamond Premier + Söhne | premium-luxury / editorial | classical, considered |

#### Palette recipes

Specific palettes per archetype that ship-ready:

**editorial:**
- bone `#F4F1E8`, ink `#0F1A2A`, accent `#D64A1A` (warm)
- bone `#F8F5EE`, ink `#1C1C1E`, accent `#2D3FFC` (cool)

**tech-minimal:**
- white `#FAFAFA`, near-black `#0A0A0A`, accent `#FF6A3D`
- white `#FFFFFF`, ink `#0D0E12`, accent `#0066FF`

**warm-indie:**
- cream `#F4F1E8`, night `#0F1A2A`, clay `#D64A1A`

**industrial-brutalist:**
- bone `#F0EDE3`, near-black `#0A0A0A`, signal `#FF4500`

**premium-luxury:**
- ivory `#F5F0E8`, oxblood `#2A0808`, gold `#B08D57`

**dark-tech:**
- surface `#0A0A0A`, text `#FAFAFA`, accent `#00E5FF`

**security-terminal:**
- surface `#0A0E0A`, amber `#FFB000`, bone `#E8E2D1`

(Etc. — generate per archetype as needed. The point: these are tested combinations, not random hex picks.)

---

The library above is fully self-contained. The skill never requires another skill to define a brand. If `brandkit`, `stitch-design-taste`, or other taste skills happen to be installed and the user wants a different opinionated voice, they can override during the brand-creation flow — but it's purely optional.

### How brand applies to prompts and hand-offs

Every image prompt or skill hand-off references the brand. Example prompt with brand applied:

```
### Image prompt

Aspect ratio: 1:1 (LinkedIn carousel slide)
Brand palette: ink #0B132B, clay #D64A1A, cream #F4F1E8
Typography (if any visible text): display in a confident grotesk, restrained
Mood: grounded, specific, no hype

A share button frozen mid-press. No confetti, no animation, no celebration —
just the moment of intent. Background empty. The button looks ready but the
system is silent. Minimal, deliberate. Use the brand palette; ink button on
cream surface, clay accent only where the press point would be.

(Paste into Nano Banana, ChatGPT image, Midjourney v7, or Gemini Image.)
```

For skill hand-offs, pass the brand JSON as context alongside the unit text. The receiving skill applies the brand within its own conventions.

---

## Update mode (auto-detected in Phase 0)

After detecting N new commits since `last_commit_hash`:

1. Pull the new commits using the same author filter from state.
2. Cluster them: do they extend an existing chapter (same theme, area), or form a new chapter?
3. Ask per cluster: "These M commits look like they belong in [chapter X, 'The Y rewrite'] / a new chapter [proposed title]. Which?"
4. Draft / interview the new content (cadence inherited from state).
5. Append to existing chapter file OR create new chapter file.
6. Update state.

No re-kickoff. No re-confirming identity. State is the source of truth.

---

## Revise mode

If the user says "revise chapter NN" / "rewrite chapter NN" / wants to redo a chapter:

1. Load the chapter from file + state.
2. Ask: "Keep existing reflection ('From me'), or re-interview?"
3. Re-read commits (resolve hashes; if stale, run rebase recovery).
4. Re-draft the technical narrative.
5. Re-weave reflection (existing or freshly interviewed).
6. Save. Update state.

---

## Rebase recovery

If `.logbook-state.json` has hashes not reachable in current `git log`:

1. Don't panic. Try to re-attach.
2. For each stale commit, search current `git log` for a `(subject + date + author_email)` match. Exact subject match first; if none, fuzzy on subject substring within ±2 days.
3. Present:

   ```
   Looks like history was rewritten since last update.

   ✓ 28 commits re-attached to new hashes
   ✗ 4 commits couldn't be matched:
     - <subject>  (in chapter 5)
     - <subject>  (in chapter 12)

   Options for the 4 lost ones:
   - Skip — leave chapter prose as-is, accept stale references
   - Replace — pick new commits to back-fill what those covered
   - Regenerate — re-draft affected chapters from scratch using current history
   ```

After the user picks, update state with new hashes. Don't auto-rewrite chapters without confirmation.

---

## When you bail out

- Not a git repo and the user can't point you at one → stop, explain that this is git-history-driven.
- Repo has fewer than ~10 commits → tell the user the format is overkill for this scale; offer a single-doc summary instead.
- User wants to abort mid-flow → save state at the current chapter and confirm: "Stopped at chapter NN. Run the skill again later and it'll resume from NN+1. State at `<output-dir>/.logbook-state.json`."

---

## Files this skill writes

| Path | When |
| --- | --- |
| `/tmp/<repo-slug>-commits-raw.md` | Phase 1 |
| `<output-dir>/.outline.md` | Phase 2 |
| `<output-dir>/.logbook-state.json` | Phase 3+ (after every chapter save) |
| `<output-dir>/chapter-NN-<slug>.md` | Phase 3+4, per chapter |
| `<output-dir>/README.md` | Phase 3+ (continuous), finalised Phase 5 |
| `<output-dir>/highlights.md` | Phase 5 |

Nothing gets committed back to the source repo. The logbook lives in its own dir.
