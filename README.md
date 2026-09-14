# claude-workflow

Skills and agents I actually use with Claude Code, plus the method behind them.

These are templates, not a framework. Copy the ones that fit, change the placeholders,
delete the rest. The method below matters more than the files.

---

## The method

### 1. Put your guardrails in the tool, not in your discipline

A rule that depends on you remembering it at 7pm on a Friday does not exist. Every rule I
care about is encoded in the skill that does the work:

- `implement-ticket` refuses to write implementation before a failing test exists
- deployment skills require an explicit, in-the-current-message approval before any
  non-development environment — a general "go ahead" from earlier does not count
- read-only agents are declared with a restricted tool list, so exploration cannot write

The agent does not need to be trusted if the procedure makes the unsafe path unavailable.

### 2. Review is two-directional

Most people automate writing code. Far fewer automate *receiving criticism* of their own.
Three skills cover the full loop:

| Skill | Direction |
|---|---|
| `implement-ticket` | I write code, test-first |
| `review-pr` | I review someone else's code |
| `respond-review` | Someone reviews mine, I process the feedback |

`respond-review` deliberately **triages** rather than complies: each comment is sorted into
*fix it* or *answer it*. Applying every reviewer suggestion mechanically is not
collaboration, it is capitulation — and it degrades code as often as it improves it.

### 3. A separate reviewer breaks the self-validation loop

The `reviewer` agent exists because the author of a change — human or model — is the worst
judge of it. Its prompt says so explicitly: *the code you are reviewing was often written
with AI assistance, you trust it by default not at all.*

It runs before the PR is opened, it reports, and it never edits. Reporting and fixing are
different jobs, and merging them is how findings get quietly "fixed" into nonexistence.

### 4. Close the loop, and abandon what fails

`self-audit` is the part I would keep if I could keep only one. Weekly, it reads my own
transcripts and:

1. **Reopens the previous recommendations first**, before analysing anything new
2. **Verifies factually** whether each was applied — by reading the file, not by trusting
   the journal
3. **Measures the effect** on this week's numbers
4. **Classifies**: `applied + effect measured` / `applied, no effect` / `not applied` /
   `obsolete`

Then the rule that makes it a loop rather than a weekly report:

> A recommendation marked "applied, no effect" twice in a row is **dropped**, not repeated.
> One marked "not applied" twice in a row: ask once, then abandon it.

Advice that does not survive contact with reality gets deleted. Without that rule, a
journal becomes a list of good intentions that grows forever and changes nothing.

### 5. Write fewer skills than you think

I have written far more skills than I use. Two of them carry the overwhelming majority of
my daily work; several have never fired once. Writing a skill costs an hour. Keeping a dead
one costs you every time you scan your own list.

This repository contains the ones that earned their place, not everything I wrote.

---

## What is here

```
skills/
├── self-audit/         weekly audit of your own Claude usage, with a closed loop
│   ├── SKILL.md
│   └── extract_usage.py    parses local transcripts into a digest
├── implement-ticket/   strict TDD: red, green, refactor — no implementation before a failing test
├── review-pr/          senior review of someone else's pull request
├── respond-review/     process review feedback on your own PR, triaging fix vs. answer
└── postmortem/         blameless incident postmortem, timeline reconstructed from evidence

agents/
└── reviewer.md         independent pre-PR reviewer, read-only, never edits

templates/
├── SKILL-template.md   skeleton with the conventions that matter
└── AGENT-template.md
```

## Placeholders

The skills came out of a real setup, with the specifics replaced. Search for these and
substitute your own before use:

| Placeholder | Meaning |
|---|---|
| `<YOUR_REPO>` | repository name |
| `<YOUR_EMAIL>` | your account on the code hosting platform |
| `<YOUR_WORK_ORG>` | string distinguishing work projects from personal ones |
| `<CLIENT>` / `<PRODUCT>` | client or product names in examples |

`review-pr` and `respond-review` target **Azure DevOps** (`az repos` CLI). The procedure
transfers to GitHub or GitLab; the commands do not.

## Installing

Skills go in `~/.claude/skills/<name>/SKILL.md`, agents in `~/.claude/agents/<name>.md`.
Both are read at session start. Invoke a skill by name (`/self-audit`) or let its
`description` trigger it.

`self-audit` reads the transcripts Claude Code writes under `~/.claude/projects/`. It reads
your own local files and sends nothing anywhere.

## Language

The skill prompts are written in French, since that is the language I work in. The
procedures translate directly; the wording is worth rewriting in yours rather than
translating — a prompt in the language you actually think in is a better prompt.

## License

MIT — see [LICENSE](LICENSE).
