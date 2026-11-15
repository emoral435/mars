# Build prompt: "Mission Control", a local cockpit for one person running a fleet of coding agents
This file is a prompt for coding agents. It describes a local web app (plus a thin macOS wrapper) that lets one
person run 5 to 10 long-lived coding-agent sessions as a team: talk to them, see their pull requests, decide, review
documents, and keep every outward action behind the person's explicit go.
It is written to be built in pieces. Read Part A once. Then build one section of Part C at a time, in the order of
Part D. Each section is self-contained: goal, behavior, data, API, acceptance tests, out of scope.
Names used here:
- **Owner**: the one human who uses the app.
- **Leader**: one long-lived agent session that relays the owner's messages to the other agents and reports back.
- **Workers**: the other long-lived agent sessions, each with one lane (for example "platform", "rollout", "reviews",
  "automations", "misc", "app"). The app's own code is maintained by one worker ("app").
- **Board**: the local web app this prompt builds. Call it whatever you like; this file says "the board".
---
## Part A: how to use this prompt
1. Work in one git repo for the board. Keep the board's live state in a separate state directory
   (`~/.board/state/` by default), never inside the repo.
2. Build in the order of Part D. Do not start a section until the previous one passes its acceptance tests.
3. For each section:
   - write 5 lines first: intent, what changes, acceptance tests, out of scope, risks;
   - write the failing tests;
   - write the code;
   - run the whole test suite with a time limit (for example `timeout 300`);
   - have a fresh agent, on a different model if you can, review the diff before you merge it.
4. Prefer the standard library. Python 3.12+ for the server, plain JavaScript modules and CSS for the front end, no
   build step, no framework. Vendor any small library you need (a Markdown renderer, a diff renderer) into
   `static/vendor/` with its license.
5. Every section that adds a write path to the outside world (GitHub, Slack, a deploy tool) needs 2 security review
   rounds and a test for every refusal case. Section C12 lists the rules.
6. Keep each function to one idea. Name things for a new teammate. Comments say why, in 2 lines or fewer.
7. Never put the owner's real data (people, repos, channels, URLs) in the repo. Tests use fixtures with made-up names.
---
## Part B: what the board is for, and its principles
### The problem
One person runs many agent sessions in terminal panes. Each agent works on its own lane, opens PRs, asks questions,
and waits for decisions. Without a cockpit, the owner loses track: questions get missed, PRs wait for a click,
agents repeat work, and outward actions (merges, posts, applies) happen without a clear go.
### Principles
1. **One place for the owner.** Every question, PR, decision, document to read, and urgent item shows in one app.
2. **Nothing goes out without the owner's go.** Agents prepare; the owner clicks. Merges, approvals, posts and
   infrastructure applies are gated, logged, and tied to exactly what the owner saw (a commit SHA, a plan id, a text).
3. **Code over judgment.** When a rule exists, code enforces it (routing, gates, schedules, skill triggers).
   Agents use judgment only where no rule fits.
4. **Read-only by default.** The board reads GitHub and local files on its own schedule. Any write goes through a
   named, tested path.
5. **Cheap to run.** Page views never call GitHub directly; a background collector caches. Polls are small and
   incremental. Messages to agents are short.
6. **Plain words.** The UI and every message use short sentences and active voice.
7. **Local and private.** It binds to 127.0.0.1, pairs each browser once, and keeps state on disk as plain files
   (TSV, JSON, JSONL, Markdown) that a person can read.
---
## Part C: the sections
### C1. Skeleton: server, state dir, pairing, static pages
**Goal.** A local HTTP server that serves the front end and a JSON API, with a paired-browser check.
**Behavior.**
- `board serve --port 8765` starts a threaded HTTP server on 127.0.0.1 only.
- First visit shows `/pair`: the server prints a one-time code in its terminal; entering it sets an HttpOnly,
  SameSite=Strict cookie. Every API call needs the cookie. "Unpair all browsers" in Settings rotates the secret.
- Every state-changing endpoint is POST, checks the cookie and an `Origin` header equal to the board's origin, and
  takes a JSON body under 1 MB.
- The page shell: a left sidebar (pages with counts, New chat, the chat list, a footer menu), a top bar (mode pill,
  Tell button, bell), and a main area.
- One `/api/poll?since=<version>` endpoint returns only what changed since a version number (chats, counts, bell,
  PR rows). The front end polls every 5 s while visible and every 60 s while hidden.
**Data.** `state/` holds every file. A small `store.py` gives atomic write (write temp, fsync, rename), append-only
JSONL with a lock, and TSV read/write.
**Acceptance tests.**
- A request without the cookie gets 401. A POST with a foreign Origin gets 403.
- An atomic write interrupted mid-way leaves the old file intact.
- `poll` with the current version returns an empty change set in under 20 ms on a fixture with 1,000 chat messages.
**Out of scope.** Auth for more than one person. Remote access.
### C2. The agent bus: talking to sessions
**Goal.** The board and the agents exchange messages through one local message bus.
**Behavior.**
- Use an existing local agent-to-agent channel if your agent harness has one (many have an "intercom" or
  "send message to session" feature). Otherwise build a tiny one: a Unix socket broker with `send(to, text)`,
  `ask(to, text) -> reply`, `list()`. Each session registers a name and an id.
- The board registers as `board`. Messages from the board start with a short tag the agents can route on:
  `[board] tell t123: <text>`.
- The leader replies with a CLI the board ships (`board-cli reply t123 <text>`), which appends to the chat and bumps
  the poll version. Agents never edit state files by hand.
- **Roster.** `state/roster.json` lists each agent: name, lane, session id, pane id (if a terminal multiplexer is used),
  aliases (old names). A message to a retired or renamed agent goes to its successor, with a note at the end.
- Delivery receipts: each send records sent, delivered, failed. A failed send shows a red "Not delivered, queued"
  bar with Retry; the text is kept either way.
**Acceptance tests.**
- A send to an alias reaches the successor and appends "(sent to <old>, now <new>)".
- A send while the broker is down records failed and shows Retry; Retry after the broker returns delivers once.
- `reply t123` with a text holding quotes and newlines round-trips exactly.
**Out of scope.** Cross-machine delivery.
### C3. Chats
**Goal.** The landing page is a chat list, like a messenger: one chat per topic.
**Behavior.**
- **New chat** creates a chat with the leader. A message ending in `?` is a question (the chat stays open); anything
  else is an instruction. Each owner message gets an id `t<N>`.
- Chats can also belong to an item: a decision (C5), a PR (`pr-<repo>-<n>`), a document (`doc-<hash>`), a command
  output (`cmd-<id>`). Each item's thread is a chat.
- **Reply length and style chips** on every compose box: a length choice (`tldr` = 1 to 3 lines, `deep` = full
  detail; picking one clears the other; neither is fine) and a separate style chip (for example a strict plain-English
  mode) that combines with either. The chips add one short line to what the agent receives.
- **Trigger phrases** (Settings): a table of phrase to skill name. When a sent message matches (case-insensitive,
  whole words), the board adds "Load and follow the <skill> skill (<path>) for this." to the relayed text, and the
  sent message shows a small chip with the skill name. Matching is code, not agent judgment.
- **Quote to reply.** Select text anywhere (a chat, a PR view, a document), press `q`: the text goes into the input as
  a Markdown quote. On a document under review, `q` means comment instead (C7).
- Markdown input: Enter sends, shift+Enter adds a line, a `- ` line continues a list.
- Archive and Close per chat. Unread dot per chat. Chat scale: render the newest 200 messages, load older on scroll.
**Data.** `state/chats/<id>.jsonl` (one message per line: id, from, ts, text, kind, refs). `state/chats/index.json`.
**Acceptance tests.**
- A message ending in `?` keeps a decision chat open; any other text closes it as the answer.
- Picking `deep` after `tldr` leaves only `deep` on.
- A trigger phrase in the middle of a sentence fires; the same letters inside another word do not.
- A chat with 50,000 messages opens in under 300 ms.
### C4. Modes: Away and Focus
**Goal.** One switch tells every agent what may go out.
**Behavior.**
- **Normal**: agents work and prepare; outward actions need the owner's go per item.
- **Away**: agents keep building locally; nothing goes out (no pushes, merges, posts, approvals, no reading the
  owner's messaging app). Urgent items wait silently in the bell. The pill shows "Away · 3h".
- **Focus**: a stated set of low-risk actions may happen in the owner's name (see C9 auto-approve rules), each logged.
  The pill shows "Focus · 42 min · 5 handled".
- Clicking the pill ends the mode and shows a digest: sent, auto-approved, skipped, queued, urgent.
- Every mode change is broadcast to all agents with the exact rules, and written to `state/mode.json`.
- **Away + detach lanes** (optional, C15): on entering Away, the board asks each worker for a handoff and starts a
  cloud run per lane.
**Acceptance tests.**
- Entering Away sends one message to each roster agent within 5 s.
- In Away, every gated write endpoint (merge, approve, post) refuses with "Away is on".
- The digest lists exactly the actions logged during the mode.
### C5. Decisions
**Goal.** A decision is a chat with buttons.
**Behavior.**
- An agent creates a decision with `board-cli decide new --title ... --options go,hold,"(a) ...","(b) ..."
  --owner <agent> --body-file <md>`. It gets an id `D<N>` and a chat.
- The chat opens with the option buttons on top, the body, the thread, and the reply box.
- A click on an option sends at once. A typed answer that closes the decision waits 10 s behind an Undo toast.
- The owner's answer goes to the decision's owner agent, to `state/answers.tsv`, and into the thread.
- **No recommendation inside the options.** If the agent has a view, it goes on a separate labeled line.
- An archive page lists every decision, open and decided, with the picked option, who, when, and what it caused
  (drafts, PRs).
**Acceptance tests.**
- Undo within 10 s sends nothing and puts the text back.
- A decision whose body names a PR can be snoozed until that PR merges, and returns when the collector sees it merged.
### C6. The PR board
**Goal.** One row per open PR the owner authored, grouped by work, with one clear next step each.
**Behavior.**
- **Collector.** A background job lists the owner's open PRs (GitHub GraphQL via the `gh` CLI) every 5 minutes and on
  Refresh: title, head SHA, base, mergeable state, review decision, required checks, labels, body, files changed.
  Cache by head SHA. Page views read only the cache.
- **Stages** (first rule true wins): Merged today; Ready to merge; Approved (one thing left, named); In review;
  Asked (the owner asked someone); Not asked. Each row shows one note and at most one button.
- **Work groups.** A PR body's first line `Part of <ID>` groups PRs. A local file `~/plan/work.md` (one line per ID:
  `ID | intent | repos | ticket | owner=<agent> | stack=<repo#n>,<repo#n>`) gives the header and order. Stack syntax:
  `,` means "based on the PR before it", `+` means "merge order only, same base", `^repo#n` means "base only, from
  another stack". A `stack sync <ID>` command rewrites each PR's `Stack: 1/3 ... -> 2/3 ...` body line and rebases
  bases to match, with `--check` for a dry run.
- **Chips on every row:** Merge when ready (on/off, one click), Hold (with a reason). Both visible, not in a menu.
- **Compose ask.** Tick rows, get the text of a review-ask template, Copy. The board never posts it; "Mark as asked"
  records it.
- **Suggest reviewers.** From the repo's CODEOWNERS for the changed files, then who reviewed those files most in 90
  days. Never the author or a bot.
- **Post-merge watch.** For 30 minutes after a merge, read the CI runs on the merge commit every 2 minutes. A failed
  deploy job is urgent. A failed policy check (for example a "deploy window" check) is a normal note. A merge into a
  non-default branch (a stack step) raises nothing.
- **Sweep** (nightly): for each stale or conflicting PR, an agent writes a verdict (drop, bump, update, keep) with
  evidence. File-overlap flags ignore a shared test file touched by 3 or more open PRs.
**Acceptance tests.**
- A PR approved on an older head counts as approved if GitHub's review decision says APPROVED.
- A stack of 3 PRs shows on one card in order, and Merge on the 2nd is refused until the 1st merges.
- `stack sync --check` on a fixture reports the exact base changes and changes nothing.
### C7. The Reader and document revisions
**Goal.** Read any Markdown file under allowed roots, comment on it, and run revision rounds with its author agent.
**Behavior.**
- Roots: `~/plan/` and `~/code/` (configurable). Text files render; Markdown renders with tables and code blocks.
- **Images.** `![alt](x.svg)` relative to the doc, or absolute under a root: `.svg .png .jpg .gif .webp`, 5 MB max,
  served through `/api/img` with a sandbox CSP (`default-src 'none'; style-src 'unsafe-inline'; sandbox`) so no
  script inside an SVG runs, even when opened directly.
- **To review** list on the Reader home: only rows an agent added with
  `board-cli review add <path> kind=decision|design|requested ref=<id> <why>`; at most 10 open; 14-day expiry.
- **Comments.** Select text, press `q`: a comment anchored by its quote, listed in a side panel. Quick chips: Agree,
  Disagree, Explain. The document never changes under the reader.
- **Revise.** "Revise with my feedback (n)" sends the ticked comments as one request and keeps the version read. The
  author agent answers each comment in the thread (`thread <key> re=c<n> mark=changed|no-change|answered <text>`),
  writes the next version, and runs `doc-version <path>`.
- **What changed.** Opening the doc after a revision shows only changed sections against the version last read, each
  naming the comments that caused it.
- **Verdict bar** at the end: Revise, Approve and implement (lists the doc's "Proposed PRs" headings as rows, then
  sends one implement message), Approve only, Park or reject.
- **Doc checker** (optional, C16): show the writing checker's result on the doc.
**Acceptance tests.**
- A comment whose quote disappears after a revision says "Text changed" and keeps the old quote.
- An SVG with a `<script>` and an `onload` renders as an image, and neither runs.
- A path outside the roots, or a `..` path, returns 404.
### C8. The bell and notifications
**Goal.** One short list of what only the owner can do now.
**Behavior.**
- Rows come from the leader (`board-cli unblock` writes a ranked list, at most 10). Each row names its trigger:
  live (something is broken now), deadline (within 24 h), or blocking (an agent has nothing else to do).
- A row leaves when its target closes (decision answered, PR merged, draft sent). **Close** removes it and records why.
- **Sign-in health** lines: for each tool the agents depend on (cloud CLI sessions, a VPN or access proxy, each MCP
  server's token file), checked every 10 minutes, read-only. Healthy: one short line. Expiring within 1 hour or
  expired: urgent, with the exact command the owner runs. The board never starts a sign-in.
- **Other updates**: non-urgent notes from agents, newest first, last 30.
- macOS notification (via the wrapper, C14) for each new urgent row. No sound in Away.
**Acceptance tests.**
- A token file whose expiry is 50 minutes away makes one urgent row, once per expiry.
- An expired session line shows the exact fix command from config.
### C9. Approvals and the review gate
**Goal.** The owner approves other people's PRs from the board, and no agent can approve on its own.
**Behavior.**
- **PR Triage page.** Paste PR links. A reviewer agent writes a digest per PR (intent, what it does, intent vs
  actual, breaking changes, security notes, findings with file:line) saved by head SHA. Each card: Approve, Comment,
  Skip. "Approve selected" for up to 10 cards.
- **The gate.** An approval reaches GitHub only through one script, `post-review --pr <url> --event APPROVE
  --commit <sha>`. Before it posts, it asks the board `POST /api/guard/consume {pr, sha}`. The board says yes only if
  the owner did a Touch ID for exactly that PR and SHA in the last 30 minutes (a one-time record in memory), and the
  SHA is still the PR's live head.
- **A shell guard** in the agent harness blocks any other command that could approve (raw `gh pr review --approve`,
  `gh api` review calls, scripts that build such a command). Block commands with `$`, backticks, globs or braces near
  those words, so an agent cannot hide one.
- **Focus auto-approve**: in Focus, one approval per head may pass with no Touch ID only when code checks all of:
  CI green; at most N changed lines (excluding docs, tests, lock files); no infrastructure, deployment or security
  files; the digest has no High and at most 2 Medium findings; the repo is not on a deny list. Logged.
- **Batch approval for dependency bots.** A decision lists bot-authored PRs at exact heads. One Touch ID arms that
  list for 24 h. Each stamp re-checks: same head, the author is the dependency bot with type Bot, the low-risk rules,
  the card still open, not stamped already. For infrastructure repos, only version pins and digests may change (no
  new resources, no plan with changes, no lock files except the infra tool's own, field by field; refuse control
  characters).
**Acceptance tests.**
- A forged state file arms nothing (the arming lives only in memory; a restart disarms).
- A head that moves between Touch ID and post is refused with 409.
- A PR outside the armed list, or by another author, is refused.
### C10. Merges, holds, and infrastructure applies
**Goal.** Merge when ready, safely, on any repo.
**Behavior.**
- **Merge when ready** chip: on with one Touch ID. Each minute the board checks: an approval GitHub counts, required
  checks green, no hold, no open human review threads, not Away, the PR's stack predecessors merged, and for
  production paths the deploy window (configurable, for example weekdays 7 PM to midnight local). Then it merges with
  `--match-head-commit` (or enqueues to a merge queue with the expected head). It reads the PR again right before the
  write and refuses a changed head or base.
- **Hold** writes a reason; the owner agent is told not to push or ask.
- **Infrastructure-as-code PRs** (for example Atlantis or a similar PR-comment workflow): read the tool's project
  config. A PR touching no project merges like any PR. A PR that touches one: turning the chip on is the go for apply
  and merge, bound to the head and the latest plan comment (by id and a digest of its text), shown in the confirm
  (projects, add/change/destroy counts). The board posts the apply comment only when the head and plan are unchanged,
  the plan succeeded, all other gates pass, and production projects are in the window. It waits for every project's
  apply success, then merges. A new head or plan before the apply: "needs your look". Around the apply: urgent note,
  no retry. State the remaining race (the apply comment cannot pin a plan) in the docs.
- Every merge and apply is a line in `state/log.md` and a note to the PR's owner agent.
**Acceptance tests.**
- A prod-path PR outside the window is refused with the next window time.
- A stack PR whose base was retargeted after the Touch ID is refused.
- A replan between tag and apply drops the chip to "needs your look" and posts nothing.
### C11. The Send queue (messages the owner posts)
**Goal.** Agents draft messages; the owner sends them.
**Behavior.**
- `board-cli draft add --to <channel or person> --text-file <md> --kind message|review|bump --asked t<N>`. A draft
  needs an `asked` reference to an owner request (or Focus on); otherwise it is refused and the text goes in the
  agent's report instead.
- Each draft shows in its chat: where it goes, the text (editable), Copy text, Send, Delete.
- **Send** (only in Focus, or with an explicit click) has one agent post exactly that text and record the link.
- Drafts expire after a set time.
**Acceptance tests.**
- A draft with no `asked` and Focus off is refused.
- Editing a draft and sending posts the edited text exactly.
### C12. Security model (applies to every section)
- Bind to 127.0.0.1. Pair browsers. Check Origin on every POST.
- Touch ID helper: a tiny Swift binary using LocalAuthentication, built at start into a fresh 0700 temp dir, its
  sha256 checked before each run. The prompt text names the exact action, item and SHA.
- Arming records live in memory only.
- Every write path has: a re-read right before the write, a pinned SHA or id, a refusal test per gate, and a log line.
- Never pass user text to a shell; build argv lists.
- File reads go through one path guard (resolve, then check it is under an allowed root; no symlink escape).
- Two security review rounds, on different models, for each write path. Fix by refusing more, never by trusting more.
### C13. Automations
**Goal.** Scheduled jobs that fire on time, need no steering, and show their state.
**Behavior.**
- **The board's own scheduler** fires every job (not an agent remembering). Jobs live in `state/automations.json`:
  id, schedule in plain words ("every weekday at 10:15 am"), owner agent, task file path, done-check, catch-up policy.
- **Task file per job**: a self-contained Markdown prompt with inputs, steps, output path, and what "done" means, so
  the agent needs no steering.
- **Template + schema jobs**: a report job (for example a weekly status) has a template file (headings, order,
  length) and a schema file (each field and its source). The owner edits both in the board. The job fills the
  template; it makes drafts only (C11).
- **Done-check**: a small command or file test per job. Retry and Run now use it, so a job that already ran says so,
  and a stuck "already queued" re-asks after 10 minutes.
- **Ack**: the owner agent must ack within 10 minutes (a busy agent acks, queues, runs next). No ack: retry once, then
  an urgent bell row.
- **Missed runs** (the board was down or the Mac slept): run once on wake if within the catch-up window, else a note.
- **Timers** are held by the board, not by agent sessions, so a restart loses nothing.
- The tab shows each job: next run, last run, result, output link, Run now, Pause, Edit schedule, Edit task, Edit
  template.
- OS-level jobs (launchd plists) show too, read-only state, with Pause/Resume.
**Acceptance tests.**
- A job due while the board was down fires once on start within its window.
- Run now on a job whose done-check passes says "already done today" and does nothing.
- A missing ack after 10 minutes produces exactly one retry and then one urgent row.
### C14. macOS wrapper
**Goal.** A small app window around the board.
**Behavior.** A WKWebView app that loads the board, opens other links in the browser, shows a Dock badge with the
bell count, and turns new urgent rows into notifications. Built with one script into `~/Applications`.
**Acceptance tests.** The badge matches the bell count within one poll. A link to an external site opens in the
default browser.
### C15. Detach to cloud runs (optional)
**Goal.** When the owner leaves, long lanes keep going on a cloud agent runner, if you have one.
**Behavior.**
- Away + detach: the board asks each worker for a handoff file (repo, branch, task, must-do, must-not-do). It waits
  18 s per lane, then keeps listening until Back.
- Each handoff must say: open PRs ready, never as drafts; no agent co-author trailers in commits; no merges,
  releases or posts.
- The board starts a run per lane through the runner's CLI and shows a trail per lane: asked, handoff received, run
  started (link), detached.
- On Back: list every run since Away began (also runs agents started themselves), with state, outcome, summary and
  PR links. Tell each owner agent to attach. A failed run is never shown green.
**Acceptance tests.** A lane whose handoff arrives late still detaches before Back. A run started by an agent
directly appears in the Back list.
### C16. Writing rules and the doc checker (optional but recommended)
**Goal.** Documents for the owner are short, concrete and checkable.
**Behavior.**
- A `writing-for-humans` skill for docs a person reads, and a `writing-for-agents` skill for handoffs and context.
- A checker script: a reader line ("For: ..."), a scope line, a budget (default 750 words, overridable with
  `Budget: N`), sections whose heading ends in "(not counted)" and a Sources section excluded, sentences of 25
  words or fewer, no semicolons, a banned-words list, captions under images, SVG only (no Mermaid) for diagrams.
- The Reader shows the checker's result on a doc.
**Acceptance tests.** Fixtures that pass and fail for each rule, including a "(not counted)" section with
subsections.
### C17. Search, keys, settings, themes
- `cmd+k`: search chats, decisions, the log, and Markdown files under the roots; an in-memory index with a size
  budget; verbs as actions (`go D12`, `open repo#12`).
- Keys: `j/k` move, `g` recommended answer, `h` hold, `d` done, `u` undo, `q` quote or comment, `/` the tell box,
  `[` sidebar, `?` docs.
- Settings: theme (light, dark, a few palettes), font, zoom, density, polls, integrations, trigger phrases, review
  ask templates.
### C18. Panes
**Goal.** Several things side by side.
**Behavior.**
- A pane holds one view: a chat, a PR, a document, or a page.
- Right-click a pane header: "Open to the left", "Open to the right". Only that pane moves.
- Drag a pane header to reorder. At most 10 panes; an 11th asks which to close.
- Keyboard: focus next/previous pane. Layout survives a reload. On a narrow screen, panes stack as tabs.
**Acceptance tests.** Moving pane 3 to the left of pane 1 changes only its position. The 11th open is refused with
a choice.
### C19. Cost and agent health
- Per agent: tokens in and out, active time, and cost from the harness's session files, for 7 and 30 days.
- Context use per session, with a warning above 70%.
- A "free up agents" view: what each agent is holding, so the leader can close lanes.
### C20. Morning brief and weekly report
- A brief of what changed overnight (merges, new PRs, answered decisions, deadlines today), opened from the bell.
  Never a pop-up.
- A weekly "what I did" job (C13) that collects the owner's merged PRs, reviews, tickets and docs into a Markdown
  file, and a report job that fills the owner's template from it.
---
## Part D: build order and milestones
1. **M1, talk:** C1, C2, C3. The owner can chat with the leader and the leader can reply.
2. **M2, decide:** C4, C5, C8 (without sign-in health). The owner can answer decisions and see urgent items.
3. **M3, see PRs:** C6 (read-only, no chips), C17 search.
4. **M4, read:** C7 with revisions, C16 checker.
5. **M5, gates:** C12, C9, C10 (merge only), C11. Two security rounds.
6. **M6, automate:** C13, C8 sign-in health, C20.
7. **M7, infrastructure applies:** C10 apply path. Two more security rounds.
8. **M8, comfort:** C14, C18, C19, C15.
After each milestone: a fresh review, a list of what is left, and a short note in `docs/HISTORY.md`.
---
## Part E: conventions for the agents that build it
- One writer per branch. Small PRs (about 400 changed lines or fewer), stacked when related, each with its own test.
- PR titles in Conventional Commits. PR body: Intent (one sentence), What changed, Risk, Rollback.
- No agent co-author trailers in commits.
- Bound every test run in time.
- Never commit secrets or real data. Tokens live outside the repo, read at runtime.
- A change to a write path ships only after its refusal tests pass and 2 review rounds agree.
- Keep `docs/GUIDE.md` describing the board as it is now; history goes in `docs/HISTORY.md`.
---
## Part F: things to decide early (write the answer in `docs/DECISIONS.md`)
1. Which agent harness and which local message bus.
2. Which terminal multiplexer, if any, and whether the board may type into panes (for restarts and renames).
3. The allowed file roots for the Reader.
4. The deploy window and which paths count as production.
5. Which repos are denied for auto-approve and batch approval.
6. Whether you have a cloud agent runner (C15) at all.
7. The default word budget for docs, and whether plans follow it.
