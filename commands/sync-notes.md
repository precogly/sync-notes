---
description: Turn a maintainer-sync transcript into GitHub records — notes posted to the Maintainer Sync Discussion, decisions inline, and from-sync issues for action items.
argument-hint: "[publish] — bare drafts into the vault; 'publish' posts the reviewed draft"
allowed-tools: Read, Write, Glob, Bash(gh issue:*), Bash(gh api:*), Bash(ls:*)
---

You are turning a Precogly maintainer sync into durable GitHub records. GitHub is
the single source of truth; the Obsidian vault is the staging area where the human
reviews before anything is published.

This command has two modes. `$ARGUMENTS` decides which:

- **bare** (no arguments) — read the sync, write a draft into the vault, post nothing.
- **`publish`** — post the draft the human has reviewed.

Never post to GitHub in draft mode. Never re-extract in publish mode. The draft
file is the handoff between them, and it is the human's to edit.

## The vault

The vault lives at `$PRECOGLY_VAULT`, defaulting to `~/Desktop/vault`. Syncs are
flat, date-keyed files under `Precogly/Syncs/`:

    2026-07-13.md              the human's rough notes
    2026-07-13-transcript.md   pasted out of Fathom
    2026-07-13-fathom.md       optional — Fathom's own action items
    2026-07-13-draft.md        yours to write, the human's to edit

Resolve the sync date by listing `Precogly/Syncs/` and taking the newest date that
has a `-transcript.md`. That qualifier matters: Daily Notes writes into this same
folder, so a bare `YYYY-MM-DD.md` with no transcript beside it is just a note the
human made on some other day, not a sync. If nothing has a transcript, say so and
stop — they haven't pasted it in yet.

## Draft mode

### 1. Read the sync

Read the notes and the transcript, plus `-fathom.md` if it exists. Then read the
repo: you will need real issue numbers, PR numbers, and GitHub handles to link
against, and this is the thing Fathom cannot do.

### 2. Extract, with the right source doing the right job

**The transcript is the record.** What was said, and where each thread landed, comes
from here. Decisions are extracted from the transcript, not from anywhere else.

**The notes are a coverage checklist, not a source of facts.** They are bullets typed
mid-meeting: fragments, shorthand, misspellings, half-words. Read each bullet as a
*fuzzy pointer* into the transcript — find what it refers to and use the transcript's
account of it. Do not quote the notes verbatim into anything public, do not treat a
bullet as a decision because it sounds like one, and do not let a typo become a
proper noun. Their real function is completeness: **every bullet must be accounted
for in the draft.** If a bullet points at something you cannot find in the
transcript, that is not a bullet to drop — it goes to *Questions* below.

**Fathom is a cross-check, never a source.** If `-fathom.md` exists, extract your own
action items first, from the transcript, without reading it. Only then diff Fathom's
list against yours, and put anything Fathom caught that you did not into *Questions*
as a question. Fathom's list is derived from the same transcript by a model that has
never seen this repo, so its errors are correlated with yours and its confidence is
not evidence. It never promotes anything into the record on its own.

**The human is the decision authority.** Not the transcript, not their notes, not you.
That authority is exercised at review time, which is why draft mode writes a file
instead of posting.

### 3. Check for a prior run

The command must be safe to run twice. Before drafting, check whether this sync has
already been recorded: list open `from-sync` issues, and read the comments already on
the sync thread. Anything already posted is skipped, and you say what you skipped.

### 4. Find the thread

Notes are posted as a comment on this week's discussion in the **Maintainer Sync**
category:

    gh api graphql -f query='
    { repository(owner:"precogly", name:"precogly") {
        discussions(first:10, orderBy:{field:CREATED_AT, direction:DESC}) {
          nodes { number title url id createdAt category { name } }
    }}}'

Filter to `category.name == "Maintainer Sync"` and match the sync date. If there is
no thread for this sync, still write the draft — just leave `thread:` empty and tell
the human, so a missing thread never costs them the extraction work.

### 5. Write the draft

Write `Precogly/Syncs/<date>-draft.md` in exactly the shape below, then stop. Tell
the human it's ready, name the file, and summarize what's in it in two or three
sentences — including anything in *Questions*, since that is what needs them most.

Post nothing. Create nothing. Ask for no approval; the file *is* the approval
mechanism.

## The draft file

```markdown
---
sync: 2026-07-13
thread: https://github.com/precogly/precogly/discussions/12
status: draft
---

# Notes

## Sync — 2026-07-13

**Present:** <names>

### TL;DR
- <2–4 bullets: the gist someone skimming a month later needs>

### Discussion
- **<topic>** — <what was said and where it landed>

### Decisions
- **<what we decided>** — <why; what we weighed and rejected>. <context link>

### Action items
- [ ] @owner — <task> {{issue:short-key}}

### Parking lot
- <deferred topics>

# Issues

## <imperative, specific title — "Add X", not "X stuff">
key: short-key
labels: from-sync, enhancement
assignee: AlvinKuruvilla

**Context:** <1–2 sentences: why this came up. From the 2026-07-13 sync.>
**Done looks like:** <the concrete outcome that closes this>
**Open questions:** <threads raised against this issue and left unsettled — omit the field if none>

# Questions

- <anything ambiguous, unresolved, or that Fathom heard and you didn't>
```

Three sections, three fates, and the human can edit any of them:

**`# Notes`** is posted verbatim as a comment on the thread. What the file says is
what gets published, byte for byte — so this section must contain nothing the human
would not want on a public repo. No transcript quotes, no raw notes, no Fathom
artifacts.

**`# Issues`** — each `##` block becomes one issue. `key:` is what the `{{issue:key}}`
placeholders in the action items point at; `labels:` always includes `from-sync`;
`assignee:` is a real GitHub handle or the literal word `unassigned`.

**`# Questions`** is for the human alone and is **never published**. This is where
uncertainty goes to be resolved by a person. A half-made decision, an unclear owner,
a bullet you could not place in the transcript, a thing Fathom heard and you did not
— all of it belongs here rather than being guessed at in Notes.

## Publish mode

The draft is now the human's text, not yours. Do not re-extract, do not "improve" it,
do not fix its prose. Publish exactly what it says.

1. **Read the draft.** If `status:` is already `published`, stop and say so — nothing
   is posted twice.

2. **Ensure the thread exists.** If `thread:` is empty, there is no discussion for
   this sync — the scheduled workflow hasn't opened one, or it hasn't run yet. Create
   it rather than failing: notes are worthless sitting in a vault.

       gh api graphql -f query='mutation($repo:ID!,$cat:ID!,$title:String!,$body:String!){
         createDiscussion(input:{repositoryId:$repo, categoryId:$cat, title:$title, body:$body}){
           discussion { number url id } } }' \
         -f repo="<repositoryId>" -f cat="<Maintainer Sync categoryId>" \
         -f title="Sync — <date>" -f body="<agenda, or a line saying notes follow below>"

   Both IDs come from the same query draft mode already runs (`repository { id }` and
   `discussionCategories`). The title format is `Sync — YYYY-MM-DD`, which is what
   draft mode matches on, so a thread created here is found by every later run.

   Write the new thread's URL into the draft's `thread:` before going further, so a
   crash between here and the end doesn't strand a thread nobody can find.

3. **Resolve the placeholders.** Every `{{issue:key}}` in Notes must match an issue
   block. An unresolved placeholder is a hard stop: the human deleted an issue block
   and left a dangling reference, and posting that would publish a broken record. An
   issue block that nothing points at is only a warning — say so and carry on.

4. **Create the issues first,** so the links resolve:

       gh issue create --title "…" --body "…" --label from-sync [--assignee <handle>]

   Each body follows the issue block and ends with a backlink to the sync thread.
   Use `--assignee` only for a real handle; never assign a name you inferred.

5. **Substitute** each `{{issue:key}}` with the real `#N`.

6. **Post the Notes section** as a comment on the thread:

       gh api graphql -f query='mutation($id:ID!,$body:String!){
         addDiscussionComment(input:{discussionId:$id, body:$body}){ comment { url } } }' \
         -f id="<discussionId>" -f body="<the Notes section, substituted>"

7. **Write back.** Update the draft in place: substitute the placeholders there too,
   set `status: published`, and record the comment URL. The vault file becomes the
   local record of what was published, and a second `publish` becomes a no-op.

8. **Report** with links to the posted notes and every issue created.

## Rules that bind both modes

1. **Faithfulness over completeness.** Never invent a decision, an owner, or a
   commitment. Uncertainty goes to *Questions*, always.
2. **The Decisions section is the decision log.** There is no second source. A
   reversal is a *new* decision line that references the one it overturns.
3. **The transcript never leaves the vault.** It is raw meeting content and
   `precogly/precogly` is public. Never paste it, quote it at length, or commit it.
4. **Bidirectional links.** Every action item links to its issue; every issue
   backlinks to the sync thread.
5. **Owners are explicit.** A real `@handle`, or the literal word *unassigned*. Never
   blank-but-implied.
6. **Dates are ISO.** The sync date is the join key across the vault, the thread, and
   the issues.
7. **Re-runnable without duplicates.** Running either mode twice must not double-post
   the notes or open the same issue again.
