---
description: Generate an interactive visual plan for an issue
argument-hint: '<issue id | issue URL | free-text ask>  e.g. ABC-123'
---

Build a single, self-contained, no-build HTML plan for **$ARGUMENTS** — the same
artifact as the template itself when filled in: a slim toolbar (plan title +
issue badge), a left panel with five tabs (Executive summary / Full plan /
Security & Risks / Open questions / Comments), a right panel listing the
fixed set of business domains as colored pills, and a middle section that
breaks the selected domain's change down by layer (UI / API / Modules /
Services / Models·schemas / Cloud). Output goes to `docs/plans/$SLUG-plan.html`
(see Step 5 for how `$SLUG` is derived). The Comments tab is not PLAN DATA —
it's populated live by the engine's own highlight-to-comment feature (select
any text in the plan to annotate it), not written by you; see Step 6.

You are **not** authoring HTML/CSS/JS. The engine is generic and already
written. Your job is to read the real issue and the real code, then produce
**only the PLAN DATA blocks**, and drop them into the template. Everything
else is copied verbatim.

## Before you use this in a real repo

`lnr-plan.template.html`'s `WORLD_MAP` constant (and its mirror,
`lnr-plan.worldmap.json`) ship with **placeholder** domains and modules
(`core`, `billing`, `notifications`, `internal-ops`) that only show the
shape of the data. Replace them with your own repo's real business domains
and modules before running this command for real — see `SETUP.md` for a
one-time prompt that has your own agent generate a first draft from your
codebase.

This command also assumes an issue tracker with fetchable issues (id,
title, description, comments) — written against Linear's tool names
(`get_issue`, `list_comments`, `get_issue_status`) below; swap in your own
tracker's equivalent calls if you use something else. If you have no issue
tracker at all, the command still works from free-text asks alone (see Step
0's fallback) — you just never get an issue badge.

## Files

- **Template (the engine):** `lnr-plan.template.html` — a complete, working
  page whose plan-data regions are fenced with `▸▸ PLAN DATA` markers:
  `HEADER_SUB` (1 HTML string) and `PAGE_TITLE`, `ISSUE_REF`, `EXEC_SUMMARY`,
  `SECURITY_RISKS`, `OPEN_QUESTIONS`, `IMPACT`, `DB_DIFF`,
  `SHARED_SCHEMA_DIFF`, `CHAIN_MERMAID`, `FULL_PLAN_MD` (9 JS blocks), in
  that order. `PAGE_TITLE` is a JS string (not an HTML fence inside
  `<title>` — that element uses raw-text parsing, so a comment fence there
  would show up literally in the real tab title instead of being stripped)
  — just the plan's short name (2-4 words), never the issue id itself; the
  engine sets `document.title` from it and prefixes `ISSUE_REF.id` at
  runtime so the tab always reads "ABC-123 - `<name>`" when there's a linked
  issue, without relying on every run to format it consistently by hand.
- **`WORLD_MAP` — fixed, NOT a plan-data block.** The domain/module registry
  (ids, display names, deliberately-picked colors) lives in `WORLD_MAP`
  inside the template and is mirrored verbatim in `lnr-plan.worldmap.json`.
  **Never add, rename, or recolor a domain or module as part of running this
  command** — those two files are edited by hand, rarely, independently of
  any plan. If the issue touches code that has no existing module to attach
  to, do not invent one: pick the closest existing module, say so in
  `OPEN_QUESTIONS`, and tell the user in your final report that `WORLD_MAP`
  may need a manual addition first.
- **Domain registry (reference, read-only, if you have one):** some repos
  keep a single source-of-truth file mapping domains to their folders/
  collections/modules (our own internal copy of this command points at
  `packages/shared/src/domains.ts`, for example). If yours has an
  equivalent, read it first for orientation and cross-check its domain/
  module ids against `WORLD_MAP` — they must match exactly (`WORLD_MAP` is
  the one that actually renders). If you have no such file, skip straight
  to the real code below.
- **Output:** `docs/plans/$SLUG-plan.html`.

## Step 0 — Resolve the issue, map the domains

- `$ARGUMENTS` is not always just an id — the template's "Send notes to
  Conductor" / "Copy notes" buttons (panel footer, shown once the viewer has
  written any risk feedback, question answer, or highlight comment) produce
  a longer prompt like `"Re-run /lnr-plan ABC-123 and update
  docs/plans/abc-123-plan.html — incorporate the notes below into the
  plan... Security & Risks\n1. <risk title>\nFeedback: <feedback>\n\nOpen
  questions\n1. <question>\nAnswer: <answer>...\n\nComments\n1. On: "<quoted
  excerpt the viewer selected>"\nComment: <comment>..."`. Extract an issue
  id or tracker URL from **anywhere** in `$ARGUMENTS`, not just when the
  whole string is one — then fetch it with your tracker's tools (e.g.
  Linear's `get_issue`, `list_comments`, `get_issue_status`). Use the
  issue's own title, description, and comment thread as the source of truth
  for `HEADER_SUB` and `EXEC_SUMMARY` — don't paraphrase away specifics. Set
  `ISSUE_REF = { id: "<identifier>", url: "<the issue's actual URL from the
  API response>" }` — never guess the URL.
- Treat any text in `$ARGUMENTS` beyond the id/URL as real input, not noise:
  answered open questions, risk feedback, highlight comments, explicit
  instructions, new constraints. A "Feedback:" line on a security risk means
  the viewer is pushing back on, confirming, or adding context to that
  risk — fold it into the risk's `note`, adjust `addressed` if the feedback
  changes it (e.g. "let's just fix it now" moves a `"no"` toward
  `"yes"`/`"partial"`), or drop the risk if the feedback says it isn't real.
  When a question that was previously in `OPEN_QUESTIONS` now has a real
  "Answer:", fold the resolution into the relevant block (`EXEC_SUMMARY`,
  `IMPACT`, etc. — wherever it actually belongs) and drop it from
  `OPEN_QUESTIONS`; keep a question there only if it's genuinely still open,
  including ones the incoming text marked `(no answer yet)`. A "Comments"
  entry is a note anchored to a specific quoted excerpt of a *previous*
  render of this plan, not to a PLAN DATA field directly — search the prior
  version of `docs/plans/$SLUG-plan.html` (or the issue's own content) for
  that quoted text to find which block produced it (an `IMPACT` item, a
  `FULL_PLAN_MD` section, `EXEC_SUMMARY`, etc.), then treat the comment the
  same way as risk feedback or a question answer: fold in a decision,
  or carry it forward as an `OPEN_QUESTIONS` entry (with `suggestions`) if
  it's genuinely a question back to you.
- If `$ARGUMENTS` has no resolvable issue anywhere in it, leave
  `ISSUE_REF = { id: "", url: "" }` (the header badge stays hidden) and build
  `HEADER_SUB` as a concise restatement of the ask.
- Launch one or more `Explore` agents (in parallel if the change spans
  modules) to map exactly which domains/modules this touches. Point them at
  your domain registry first for orientation if you have one, then the real
  code — your backend's feature folders, your frontend's route folders,
  your shared schema files, whatever your repo's own layout actually is.
  Demand real artifacts, quoted with `file:line`: endpoint method+path,
  collection/table/field names, service method names, schema file names.
  Every domain and module id you use in the PLAN DATA blocks below must
  already exist in `WORLD_MAP` — if an Explore agent finds code that
  doesn't map cleanly to one, treat it as a gap (see the `WORLD_MAP` note
  above), not license to make one up.

## Step 1 — `IMPACT`

For every domain this plan actually touches:

```js
domains: {
  "<domainId>": {
    ui:       { touched: true|false, items?: ["short plain-text line", ...] },
    api:      { touched: ..., items?: [...] },
    services: { touched: ..., items?: [...] },
    cloud:    { touched: ..., items?: [...] }
  }
},
modules: {
  "<moduleId>": "changed" | "new"   // only for modules this plan touches
}
```

- Omit untouched domains entirely — a domain is "impacted" purely by being a
  key in `IMPACT.domains`.
- Each item is a short, real line — `"POST /cycles/:id/approve (new)"`, not a
  paraphrase — either as a plain string, or as `{ text, description }` when
  what it actually does isn't obvious from the name alone, e.g.
  `{ text: "POST /cycles/:id/approve (new)", description: "Manager-only;
  flips status to pending_approval and notifies via NotificationService." }`.
  Use a description when it adds real information; skip it for anything
  self-explanatory. Omit `items` (or leave it empty) when a layer is touched
  but not worth detailing.
- **Never** set a `modules` or `schemas` key inside a domain entry — the
  engine derives both from `modules` (above) and from `DB_DIFF` /
  `SHARED_SCHEMA_DIFF` (below) so they can't drift out of sync with the data
  that already drives them.

## Step 2 — `DB_DIFF` and `SHARED_SCHEMA_DIFF`

- `DB_DIFF[domainId]`: one entry per collection/table this plan changes, read
  from the real schema — `status: "new"|"changed"`, `before` (`null`
  for a brand-new collection/table) and `after` field/index lists, verbatim
  field names and types, `diff: "added"|"changed"` on what actually moved.
- `SHARED_SCHEMA_DIFF[domainId]`: one entry per shared-schema file this plan
  changes — `{ file: "invoice.ts", summary: "..." }`. Most of these files
  mirror one business domain's own schema (e.g. `invoice.ts` → `billing`,
  `user.ts` → `core`) — list the change under that domain. A handful of
  files are usually cross-cutting and belong under `internal-ops` instead —
  shared config/env, id helpers, date/money utilities, a shared domain
  registry if you have one.
- Either one being non-empty for a domain is what lights up its "Models /
  schemas" layer — no separate flag needed.

## Step 3 — `EXEC_SUMMARY`, `SECURITY_RISKS`, `OPEN_QUESTIONS`

- `EXEC_SUMMARY.what`: 1-2 sentences, this issue's actual ask, not a generic
  template sentence. `.impact`: 1-2 sentences naming the real domains/layers
  touched.
- `SECURITY_RISKS`: real risks surfaced while exploring the code and the
  issue thread — auth/permission gaps, data exposure, migration/backfill
  needs, breaking changes. `severity` is `"high" | "medium" | "low"`. Set
  `addressed` on every risk — `"yes"` this plan's own changes fix it,
  `"partial"` this plan reduces it but a real gap remains (say what, in
  `note`), `"no"` the plan surfaces it without fixing it (tracked elsewhere,
  or it's one of the `OPEN_QUESTIONS`). Be honest here — a plan that raises a
  risk it doesn't fix is more useful than one that quietly omits it. Leave
  the array empty when there genuinely are no risks (the engine renders "No
  new risks identified" and hides the tab's badge) — never pad it to look
  thorough.
- `OPEN_QUESTIONS`: genuine unresolved ambiguities — pull disagreements or
  unanswered questions straight from the issue's comment thread where
  present, plus anything you couldn't resolve from the code alone. Each item
  is a short plain-text question, or `{ text, suggestions }` — see below.
  Empty when there are none.
- **On a re-run with incoming feedback/answers** (see Step 0), decide per
  item whether the viewer made a decision or asked you something:
  - **Decision** ("go with X", "let's do Y", "flip it to false"): resolve it
    as Step 0 already says — fold it in, drop the question from
    `OPEN_QUESTIONS` / update the risk's `note`/`addressed`.
  - **Question back to you** ("what do you think", "any suggestions",
    "should we...?", or anything else genuinely asking for your take rather
    than stating one): don't resolve it. Keep the risk in `SECURITY_RISKS`
    (or the question in `OPEN_QUESTIONS`, now as `{ text, suggestions }`) and
    set `suggestions = { mode, options }`:
    - `options`: your real recommendation(s) as short, concrete, pickable
      strings — not a restatement of the question, not vague ("consider X")
      but an actual answer ("Flip the default to false now"). One option is
      fine; list more only when there genuinely are distinct reasonable
      paths.
    - `mode`: `"single"` when the options are mutually exclusive (picking
      one rules out the others — the common case), `"multi"` when more than
      one could apply at once (e.g. "which of these routes need the
      limiter?").
    - The engine renders these as radio buttons (`"single"`) or checkboxes
      (`"multi"`) in the same card the viewer wrote in, each wired to fill
      the viewer's own answer box when picked, plus an always-present "Type
      my own answer" choice — you never add that option yourself, and you
      never write prose instead of `options` when you have a real
      recommendation to offer.
  - When unsure which it is, treat it as a question and answer with
    `suggestions` — safer than silently resolving something the viewer was
    actually still asking about.

## Step 4 — `CHAIN_MERMAID` and `FULL_PLAN_MD`

- `CHAIN_MERMAID`: this plan's chain diagram as mermaid flowchart source —
  left-to-right `flowchart LR`, labelled edges, async hops (queue/Lambda/
  worker) dashed, no code fence in the string.
- `FULL_PLAN_MD`: the rest of the plan as markdown — Architecture diagram,
  Endpoints table, Data/schema impact, Code level (files) tree, Client (n/a
  if untouched), Change list table, Risks and open questions (restate
  `SECURITY_RISKS`/`OPEN_QUESTIONS` in prose here too — this is the complete
  printable document, the tabs are just a UI convenience), Verify. If you
  keep your own plan-format convention (a house style for section order/
  content), follow it here; otherwise the section list above is a
  reasonable default. Give every domain-specific section a
  `"## <Name> {#<domainId>}"` heading using the exact `WORLD_MAP` domain id,
  so "Jump to this domain in the full plan" lands on it.

## Step 5 — Assemble

- `$SLUG`: the issue id lowercased (`ABC-123` → `abc-123`) when
  `ISSUE_REF.id` is set, otherwise a short kebab-case slug of `HEADER_SUB`.
- Create `docs/plans/` if it doesn't exist. Copy the template to
  `docs/plans/$SLUG-plan.html`, then replace **only** the 10 fenced
  `▸▸ PLAN DATA` regions with your content. Leave `WORLD_MAP` and everything
  else in the engine byte-for-byte untouched. Keep the fence comments.

## Step 6 — Verify

- `node -e` compile-check the `<script>` block (extract it and run it through
  `new Function(...)`) to catch syntax errors.
- Open the file in a browser (whatever browser-automation tool you have —
  a headless-browser skill, Playwright, or just open it by hand) and
  confirm: the issue badge (if any) shows the right id and links out in a
  new tab; the left panel's five tabs switch correctly and the Security &
  Risks / Open questions / Comments badges show the right counts (and are
  hidden at zero); the right panel's domain pills show a change-badge only
  on impacted domains; clicking an impacted domain renders the right layer
  breakdown, with "Models / schemas" showing both the shared-file list and
  the DB diff cards when both are present, and any `{ text, description }`
  item showing its description as a muted second line; "Jump to this domain
  in the full plan" opens the Full Plan tab and scrolls to the right
  anchor; the chain diagram's expand-icon opens it full-viewport (clones
  the rendered SVG, scales cleanly since it's vector), closes on the × or
  Escape. Each `SECURITY_RISKS` item shows its `addressed` pill and its own
  feedback box (when `SECURITY_RISKS` is non-empty); each `OPEN_QUESTIONS`
  item has its own answer box. Any `suggestions` you set render as radio
  buttons (`"single"`) or checkboxes (`"multi"`) inside that same
  risk/question card, between the note/question and the feedback box,
  always with an extra "Type my own answer" choice the engine adds
  itself — picking an option fills the feedback/answer box with it (you can
  still edit afterward); none of this lives off in `EXEC_SUMMARY` or
  anywhere else.
- **Every free-text box commits on blur or Cmd/Ctrl+Enter, not on every
  keystroke** — typing into a risk-feedback box, a question-answer box, or
  the highlight-comment box (below) does not persist or update the footer/
  badges until focus leaves the box (or the viewer hits Cmd/Ctrl+Enter).
  Confirm this: type into a box, confirm nothing changed yet, then blur it
  (or Cmd/Ctrl+Enter) and confirm it's now saved (reload the page and
  re-check the value survived, since persistence is via localStorage).
  Picking a `suggestions` radio/checkbox is the one exception — that commits
  immediately, since picking an option is already a single discrete action,
  not continuous typing.
- **Highlight-to-comment**: drag-select any text inside the main domain
  detail panel or inside the Executive summary / Full plan / Security &
  Risks / Open questions tabs (not inside another note-box, and not a plain
  click with no drag) and confirm a floating box appears near the selection
  with a quoted preview of what you selected, a comment textarea, and a
  disabled "Add comment" button that enables once you type something.
  Confirm Escape or the Cancel button dismisses it without saving, and
  clicking anywhere outside it does the same. Submitting (button, or
  Cmd/Ctrl+Enter) adds a new card to the Comments tab with the quoted text
  and your comment (plus a "Remove" button), bumps that tab's badge, and
  reveals the panel footer the same way a risk/question note does.
- The panel footer (hidden until the viewer has written at least one risk
  note, question answer, or highlight comment) shows "Copy notes" and,
  whenever `ISSUE_REF.id` is set, "Send notes to Conductor" (opens that
  issue's Conductor workspace via `conductor://prompt=...&linear_id=...` —
  see https://www.conductor.build/docs/reference/deep-links if you use
  Conductor; otherwise this button is dead weight and you can remove it
  from the template, or just ignore it). When `ISSUE_REF.id` is set, both
  buttons' text leads with "Re-run /lnr-plan `<id>` and update
  docs/plans/`<slug>`-plan.html" so pasting it (or sending it via Conductor)
  actually drives a re-plan instead of just handing over raw text. The
  header's "Approve Plan" button (left of the Domains toggle, shown only
  when `ISSUE_REF.id` is set) sends a fixed "I approve the plan, begin
  implementation" prompt the same way, after a native confirm() — unlike
  the notes buttons this is meant to kick off real implementation work, so
  needs no PLAN DATA of its own, just confirm it's wired and gated the same
  way. It also closes the tab shortly after (`window.close()`, after a
  short delay to let the `conductor://` handoff fire first) — browsers only
  let script close a tab *they* opened, so on a tab the viewer navigated to
  directly this is a no-op (you'll see a console warning, not an error —
  that's expected, not a bug to chase).
- Spot-check 2-3 claims (a field name, an endpoint path) against source so
  the plan stays trustworthy — the whole point of quoting real code instead
  of paraphrasing it.
- Report the output path. If you found a gap in `WORLD_MAP` (Step 0), say so
  explicitly rather than burying it in `OPEN_QUESTIONS` alone.

## The taste (what makes this plan good)

Every domain/module id traces to the fixed `WORLD_MAP` — never invented,
never recolored per-plan. "Modules" and "Models/schemas" layers are always
derived, never hand-authored, so they can't disagree with the data that
backs them. Risks and open questions are real findings, not padding — an
empty array is a better plan than a fabricated entry. `FULL_PLAN_MD` is the
complete, self-sufficient document; the tabs and domain pills are a reading
aid on top of it, not a replacement for it. Match that bar.
