# commit-studio

A portable agent skill that turns a git repo's commit history into a finished artifact in your voice. Pick what you're making: a long-form logbook, a social post or carousel (LinkedIn, Instagram, X), a conference talk script, a personal journal, or a custom format you describe. Same source material, different finished thing.

Built on the open [agent skills](https://skills.sh) format, so it installs into any of the 55+ supported runtimes (Claude Code, Cursor, Codex, OpenCode, and many more) via the `skills` CLI. The workflow was authored and tested in Claude Code, which uses tool conventions like `AskUserQuestion` and sub-agent spawning; other runtimes may need light adaptation if their tool surfaces differ.

It reads your commits, asks what you're making, scores them for fit against that format, writes a calibration draft so you can confirm voice and angle before the rest of the work, then drafts the artifact and runs short interviews to preserve your voice without paraphrasing. Auto-detects who "you" are across multiple commit emails. Treats AI-tool commits (v0, lovable, bolt, cursor-agent, copilot) as yours if you were driving. Survives rebases. Resumes from where it left off.

## What you can make

| Format | Unit | Length | Best for |
| --- | --- | --- | --- |
| **Long-form logbook** | Chapter | 30-60 chapters, 3-8 paragraphs each | Portfolio depth, telling the full story |
| **Social posts and carousels** | Post / slide | 200-600 words per post; carousels of 5-10 slides | LinkedIn, Instagram, X — sharing one big moment at a time |
| **Conference talk script** | Beat | 10-15 beats for a 20-30 min talk | Speaking, rehearsing, story arcs |
| **Personal journal** | Entry | 50-500 words, can ramble | Just for yourself, low-stakes, no audience |
| **Custom** | You decide | You decide | CV addendum, blog series, internal docs, anything else |

## What it looks like (book format example)

Every long-form chapter has the same shape: technical narrative grounded in real commits, then a short reflection section in your own voice. Other formats keep this shape but adjust length, voice, and structure to fit.

```markdown
## Chapter 14. The referral loop that doesn't trigger on clicks

Most referral UI rewards the share. Hit "invite a friend," confetti plays,
the system marks a "successful referral" the moment the link is clicked.
The dashboard becomes a celebration of nothing real.

Mine is gated on actual signups. `a91c4f2 feat: gate referral confetti
on real signup` introduces the loop: a share-button component handles
the copy/share flow and the post-signup celebration, but the celebration
only fires when a referred user completes the signup form. Two columns
track who-referred-whom and the unlock moment, and the internal alerts
only ping once a real account exists on the other end.

The temptation was to celebrate the share itself. Confetti at the
moment of intent is cheap dopamine and inflates the perceived loop
strength. Saving the moment until a real signup means most invitations
never produce confetti. That was the right call: when it does fire,
it means something.

### From me

> The first sketch had a confetti burst on click of the share button.
> I deleted it the same day. If we're rewarding the gesture instead
> of the outcome, we're just every other product. The whole point of
> saving the celebration for actual signup was that the loop has to
> earn it.
```

Output goes to `~/<repo-slug>-studio/` with unit files plus a generated `README.md` indexing the artifact. Talk and book formats also produce a `highlights.md`.

## Install

**Via the [`skills`](https://github.com/vercel-labs/skills) CLI** (works for any of the 55+ supported agent runtimes — Claude Code, Cursor, Codex, OpenCode, and more):

```bash
npx skills add mauritsdemuer/commit-studio
```

**Or directly into Claude Code:**

```bash
git clone https://github.com/mauritsdemuer/commit-studio ~/.claude/skills/commit-studio
```

No dependencies, no config, no API keys beyond what your agent already needs.

## Use it

From inside the git repo you want to chronicle:

```
/commit-studio
```

Or just say what you want. The description-match triggers on:

- "make a portfolio writeup of my commits"
- "tell the story of what I built here"
- "give me a dev logbook from this repo"
- "make a LinkedIn carousel from my work"
- "a talk script from this codebase"
- "a personal journal of what I shipped"

The skill shows the repo path it's about to scan as part of its first message, so a wrong-directory invocation gets caught immediately — just say "wrong repo, try /path/to/the/right/one" and it switches. After that it auto-detects your identity, asks what you're making, then a handful of scoping questions. No identity setup required.

## The flow

| Phase | What happens |
| --- | --- |
| **0. Detect** | Scans the repo. Clusters commit-emails that look like the same person. Surfaces AI-tool bots and asks if you were driving. Asks what you're making (format + platform if social). Picks up where it stopped if a state file exists. |
| **1. Corpus dump** | Filters the log to in-scope commits. Reports count, dates, themes. |
| **1.5. Shortlist** | For narrow formats (social, talk, custom): scores commits for fit and proposes 3-8 hero commits. You trim, swap, or approve. Skipped for book and journal. |
| **2. Outline** | Proposes the unit clusters (chapters / posts / beats / entries). Waits for your sign-off; cheap to reshape here. |
| **2.5. Calibration draft** | Writes ONE unit, shows it to you. You confirm voice / depth / angle before the rest of the work starts. Saves the cost of redoing 50 units if the voice was off. Skipped for journal (low-stakes, self-directed). |
| **3+4. Draft + interview** | Per unit: drafts the narrative, asks 1-5 reflection questions, weaves your answers in without paraphrasing. Saves. Next unit. |
| **Visual companion** | For social and talk formats: asks once if you want visuals. Either hands off to a detected image skill (Figma, Canva, frontend-design, lottie, gpt-image, etc.) or appends ready-to-paste prompts for ChatGPT / Nano Banana / Midjourney / Gemini Image. |
| **5. Assembly** | Format-specific wrap: index README, `highlights.md` (book/talk), `posting-order.md` (social), `script.md` (talk), or whatever the custom format calls for. |

You can stop after any unit and resume later. The skill writes its state to `.logbook-state.json` after every save.

## What's smart about it

- **Identity detection across emails.** If you've committed under `you@gmail.com`, `you+gh@gmail.com`, `12345+you@users.noreply.github.com`, and `you@work.co`, the skill clusters them as one person and confirms with you in one tap.
- **AI-tool commits as yours.** If half your repo was built through v0, Lovable, Bolt, Cursor, or Claude Code, those commits are conceptually you if you were driving the prompts. The skill asks; if yes, it folds them into the corpus and writes diff-led narratives since the commit messages tend to be terse.
- **AI-framing as a single decision.** Asked once at kickoff (only if the corpus actually has AI co-author trailers): how do you want AI involvement framed across the artifact? Don't mention / acknowledge once in the README header / mention in every unit / signal-based. Applied uniformly so you don't get pestered per unit.
- **Branch-aware.** If your work touches multiple long-lived branches (`main` + `production`, or active feature branches that never merged), the skill asks whether to include everything, just `main`, "what shipped" only, or a custom set. Solo dev on `main`? Skipped silently; no extra question.
- **Narrative arcs, not commit-per-unit.** Every format (book, social, talk, journal, custom) clusters commits into *stories* first. The unit you write — chapter / post / beat / entry — is built from a cluster of related commits with a shared tension and payoff. Writes things people want to read, not changelogs read aloud. Each arc has an anchor commit, supporting commits, an arc title (the story, not the area), a one-sentence tension, and a one-sentence payoff.
- **Storytelling structure baked in.** Every unit follows a five-beat arc (hook → setup → tension → specifics → payoff), dialled to the format. Social posts compress all five into 200-600 words; book chapters spread across 3-8 paragraphs; talk beats become the talk's own arc. Anti-cringe rules block "X lessons from Y" framing, manufactured-tension hooks, engagement bait, hashtag spam, and emoji-decorated headlines.
- **Calibration draft.** Before drafting 50 units, the skill writes ONE, hands it to you, and asks if it's the right direction. Recalibration happens cheaply on one unit instead of expensively on the whole artifact.
- **Reflection as an explicit choice.** Pick interviews per-unit, batched, single end-pass, or no reflection at all. The skill doesn't force the interview format if you don't want it.
- **Visual companion that uses what your runtime can actually do.** For social and talk formats, the skill probes the current runtime for native image generation (Codex / Gemini CLI / Claude Code with image MCP / etc.), installed image-gen skills (`gpt-image`, `nano-banana`, `imagegen-frontend-*`, `brandkit`, `lottie-animator`), and installed design skills (`frontend-design`, Figma MCP, Canva MCP). If any are available, "generate directly" or "design as code" becomes the recommended option — the skill produces real image files or designed mockups, not paste-ready prompts. Prompts are the fallback when nothing else is available.
- **Brand-style as a separate choice, fully self-contained with deep treatment baked in.** When visuals are requested, the skill asks once: pull brand from the repo (auto-detects `brand.md` / `STYLE.md` / `tailwind.config` / CSS custom properties / token files), use a file you point to, build interactively, or let the skill propose one. The skill ships with a **deep built-in taste library** — twelve named archetypes (editorial, tech-minimal, warm-indie, industrial-brutalist, premium-luxury, dark-tech, documentary, sharp-modern, cultural-zine, soft-modern, maker-craft, security-terminal), per-archetype deep guidance (visual qualities, specific fonts, real brand references, lean-into / avoid notes), categorised anti-AI-slop rules (color / layout / typography / illustration / composition), type-pairing recipes, ship-ready palette recipes, and a **brand-board image-prompt generator** that produces a one-image visual summary of the resolved brand. The library is the deep treatment by default — no other skill required. Installed taste skills (`brandkit`, `stitch-design-taste`, `high-end-visual-design`, `minimalist-ui`, `industrial-brutalist-ui`) become a purely optional override if you prefer their voice. The resolved brand gets baked into every prompt and skill hand-off so visuals feel like one identity.
- **Update mode.** Re-running after new commits land doesn't restart from scratch. It detects what's new, asks where the new work belongs (new unit or extend an existing one), and only writes the new content.
- **Rebase recovery.** Rewrote history? The skill stores each commit by `hash + subject + date + author_email`. When hashes go stale it re-attaches by the other three.
- **Voice preservation.** Reflection sections quote your answers with light punctuation tidy only. The skill won't expand "yeah it was the wsl thing" into "It was a fascinating discovery about WSL's path-handling behaviour."
- **Pause and resume.** Stop after any unit; the state file lets the skill pick up exactly where it left off next session.

## Honest limits

- **Niche.** Most useful if you've shipped substantial work and want to articulate it. Not a fit for a hobby repo with 20 commits.
- **Long-form is slow.** Per-unit interviews for 50 chapters is a multi-hour session. The skill supports batched and end-pass cadence if you'd rather defer. Social and talk formats are much faster (3-8 units total).
- **Quality scales with commit hygiene.** Garbage commit messages plus shallow diffs make harder narratives. The skill mitigates by inspecting diffs, but it isn't magic.
- **Voice override.** If your repo doesn't have a `CLAUDE.md` / `STYLE.md` / brand guide, the skill defaults to a clear, grounded voice. Override in the kickoff if you want something different.
- **Visuals are bring-your-own.** The skill detects image-capable skills you've installed or gives you prompts. It doesn't ship its own image generator.

## FAQ

**What if I work in a team?** Co-authors get named in prose where context demands it ("the backend lead built the matching pipeline; this chapter is about the UI I layered on top") but their commits stay out of analysis.

**What if I use Cursor / Lovable / v0 / Claude Code?** Those commits are attributable to you if you confirm during identity detection. The skill notices you're a vibe-coder and adjusts: lean on diff inspection over commit-message rhetoric.

**What if I want fewer / more units?** Set granularity in the kickoff. Defaults scale to format and commit count.

**What if I want to revise a unit later?** `/commit-studio revise chapter-14` (or post-3, beat-2, whatever the unit is). Or just say "rewrite chapter 14."

**What if I want technical-only output, no reflections?** Pick "no reflection" in the kickoff. The skill skips the interview phase entirely.

**What if I have multiple branches?** If only one branch has unique commits by you (the common case), the skill just uses everything. If multiple branches have your work, it asks: all / main only / "what shipped" only / custom.

**Can I run the skill multiple times with different formats from the same corpus?** Yes. The state file remembers the format, so each format gets its own output directory (`~/<repo>-studio-book/`, `~/<repo>-studio-linkedin/`, etc.) Same corpus, different finished artifacts.

**What if there's no git history at all?** The skill will tell you honestly. Different tool needed.

## Output structure (varies by format)

**Book:**
```
~/myrepo-studio-book/
├── README.md                              # Index + abstracts
├── highlights.md                          # 8-12 standout commits
├── chapter-01-...md ... chapter-NN-...md
├── .outline.md                            # Phase 2 outline
└── .logbook-state.json                    # Resume + rebase recovery state
```

**Social (LinkedIn carousel example):**
```
~/myrepo-studio-linkedin/
├── README.md
├── posting-order.md
├── post-01-...md ... post-NN-...md        # Each with optional image prompt
└── .logbook-state.json
```

**Talk:**
```
~/myrepo-studio-talk/
├── README.md
├── script.md                              # Full talk in order
├── highlights.md
├── beat-01-...md ... beat-NN-...md
└── .logbook-state.json
```

**Journal:**
```
~/myrepo-studio-journal/
├── README.md                              # Date index
├── entry-YYYY-MM-DD-...md ...
└── .logbook-state.json
```

Nothing gets committed back to the source repo. The artifact is yours, lives in its own directory.

## Worked example

[Link to a public example here. Recommend publishing your own first run as the reference.]

## License

MIT.

---

Built by [Maurits De Muer](https://www.linkedin.com/in/mauritsdemuer). If you ship it on something interesting, send the link.
