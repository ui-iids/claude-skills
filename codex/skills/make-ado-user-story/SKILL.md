---
name: make-ado-user-story
description: Write an Azure DevOps (ADO) User Story that exactly follows the University of Idaho OIT work-item standards. This means the "As a [user], I want [goal] so that [reason]" description, Gherkin (SCENARIO/GIVEN/WHEN/THEN) acceptance criteria, R.I.C.E. scoring, .5-Fibonacci story points, Category, and a parent Feature link, plus a Technical Documentation section (table of contents, schemas and tables with a data dictionary, models, endpoints, services, and implementation steps) detailed enough to hand the work to another developer. Saves the story as a paste-ready markdown file and, after a preview and confirmation, can create it in Azure Boards with `az boards`. Use when the user invokes `/make-ado-user-story`, asks to "write a user story", "make an ADO user story", "create a work item for this", "turn this into a user story", "write acceptance criteria for this story", or similar.
---

# Make ADO user story

You are writing an Azure DevOps User Story to the University of Idaho OIT
standard. The deliverable is a single markdown document that a Product Owner
can paste into Azure Boards (or that you create there for them), and that a
developer who was never in this conversation could pick up and fully implement.

There are two halves, and both are required:

1. **The work item itself.** Title, Description, Acceptance Criteria, R.I.C.E.,
   Story Points, Category, Area, Iteration, and Related Work, all formatted
   exactly as the standard below defines.
2. **Technical Documentation.** The schemas, tables, models, endpoints,
   services, and step-by-step implementation detail needed to hand off the work.

## Non-negotiables

- **Follow the standard exactly.** Field names, allowed values, scales, and
  formats come from the reference sections below. Don't invent new categories,
  states, or point values.
- **Never invent business facts.** Reach, Impact, Confidence, deadlines, the
  user role, Area, and the parent Feature come from the user (or from evidence
  in the session/codebase). If you don't know a value, ask. If the user
  doesn't know either, write `TBD — confirm with PO` and list it under Open
  Questions. A made-up R.I.C.E. score is worse than a missing one.
- **Never invent technical facts.** Column names, types, endpoints, and config
  keys come from reading the actual codebase when one is present. Proposed
  *new* items are fine, but label them **(new)** so they can't be mistaken
  for existing ones.
- **Preview before any write to ADO.** Show the full document and the exact
  `az` commands, and create nothing until the user says yes.
- **New stories start in State `New`.** Don't set any later state unless the
  user tells you to.
- **One story per sprint-sized unit.** If the work won't fit in one sprint
  (8+ points, or several independent outcomes), stop and propose a split
  before writing.

## Procedure

### 1. Gather inputs

Mine the session first: the code you've been working on, the user's
description, any linked tickets or PRs. Then ask for whatever is still missing
**in one batch** (offer concrete options where they are known):

| Input | Notes |
|---|---|
| User role + goal + reason | Who benefits, what they need, why. Drives the description. |
| Area path | e.g. `Development\Business Systems`. Required. |
| Iteration path | Default `Development` (the backlog) unless the user names a sprint. |
| Parent Feature | Work item ID + title. Every story must belong to a Feature. |
| Reach / Impact / Confidence | Pick from the R.I.C.E. tables. Ask if unknown. |
| Category | One of the five categories. Infer it, then confirm. |
| Constraints / assumptions | Deadlines, compliance, dependencies, out-of-scope items. |
| Tags | Optional. Match existing team tags if known. |

If there is **no parent Feature**, say so. Suggest a one-line Feature title
and description for the PO to create, and leave the Parent Link as
`TBD — Feature needed`. Don't author the full Feature yourself.

If the request is really a **Bug** (a failure to meet an established
requirement, with expected vs. observed behavior), tell the user. Bugs follow
a different template (Repro Steps, Waiting RCA, Priority) that this skill
doesn't produce. Ask whether to write it as a story anyway. Note that
per U of I guidance, a process change that was never communicated to
developers is an **Enhancement Request** story, not a bug.

### 2. Investigate the codebase (when one exists)

Before writing technical documentation, read the relevant code: models/ORM
classes, migrations, route/controller files, service modules, config and env
templates, and existing tests. Capture real names, types, nullability,
defaults, constraints, and current behavior. That material is the basis of
the Technical Documentation section.

### 3. Draft the story fields

Write each field per the **Field reference** below. Then:

- Compute R.I.C.E. and **show the math**:
  `(Reach × Impact × Confidence) / Story Points`.
- Estimate Story Points from the scale, and say in one line why you chose that value.
- Write the Acceptance Criteria in Gherkin per the **Acceptance Criteria
  standard** below.

### 4. Write the Technical Documentation

Follow the **Technical Documentation standard** below.

### 5. Run the quality gate

Before showing anything, check every item and fix failures:

- [ ] Description uses "As a [user], I want [goal] so that [reason]."
- [ ] Story fits in one sprint (≤ 5 points preferred. 8 means propose a split).
- [ ] Every AC scenario has GIVEN / WHEN / THEN, and every THEN is observable
      and verifiable.
- [ ] AC cover the happy path, at least one error/invalid path, and relevant
      edge cases (permissions, empty data, limits).
- [ ] AC say *what* the system does, not *how* it is implemented.
- [ ] R.I.C.E. inputs come from the allowed values and the math is correct.
- [ ] Category is one of the five allowed values.
- [ ] Parent Feature is linked (or flagged `TBD`).
- [ ] Every table/schema change has a column table **and** a data dictionary.
- [ ] Handoff test: could a developer who wasn't here implement this from the
      doc alone, without asking you anything?
- [ ] The doc is no longer than it needs to be. Remove sections that don't apply.

Answer the refinement questions the team asks at Backlog Refinement, and put
any "no" in Open Questions: *Is the story clear? Can it be completed in a
sprint? Are the acceptance criteria testable?*

### 6. Save the markdown document

Write it to `docs/work-items/<kebab-case-title>.md` in the current repo (create
the folder if needed), or to a path the user chooses. If there's no repo, use
a temporary directory and give the full path. Use the **Output template**
below exactly. Show the user the document (or a summary plus the path if it's
long), then ask: **"Create this User Story in Azure DevOps now?"**

### 7. (Optional) Create the work item in ADO

Only if the user says yes.

**a. Check the tooling.**

```bash
az extension show --name azure-devops --query version -o tsv
az account show --query user.name -o tsv
az devops configure --list
```

If anything is missing, ask the user to run `az extension add --name azure-devops`,
`az login`, or `az devops configure --defaults organization=https://dev.azure.com/<org> project=<project>`.
Never ask for a PAT in chat.

**b. Discover field reference names.** R.I.C.E., Reach, Impact, Confidence,
Category, and the U of I custom states are defined by the org's process
template, so their reference names (usually `Custom.<Name>`) vary. Look them
up instead of guessing:

```bash
# Fields on the User Story type in this project
az devops invoke --area wit --resource workItemTypesField \
  --route-parameters project=<project> type="User Story" \
  --http-method GET --api-version 7.1 \
  --query "value[].{name:name, ref:referenceName, required:alwaysRequired}" -o table

# Or inspect a sibling story / the parent Feature to see populated fields
az boards work-item show --id <existing-story-id> --query fields -o json
```

Map each document field to its reference name. Standard ones are:

| Document field | Reference name |
|---|---|
| Title | `System.Title` |
| Area | `System.AreaPath` (`--area`) |
| Iteration | `System.IterationPath` (`--iteration`) |
| Description | `System.Description` (HTML) |
| Acceptance Criteria | `Microsoft.VSTS.Common.AcceptanceCriteria` (HTML) |
| Story Points | `Microsoft.VSTS.Scheduling.StoryPoints` |
| Tags | `System.Tags` (semicolon-separated) |
| Reach / Impact / Confidence / R.I.C.E. / Category | org-specific, so use what discovery returned |

If a field can't be found, leave it out of the command and tell the user which
fields they must fill in by hand.

**c. Convert to HTML.** Description and Acceptance Criteria are rich-text
(HTML) fields. Convert the markdown with `pandoc -f gfm -t html` if
available, otherwise with Python's `markdown` package, otherwise by hand.
Render Gherkin keywords in bold, with each `GIVEN`/`WHEN`/`THEN`/`AND` on its
own line (`<br>`), so they stay readable in Boards.

**Where the Technical Documentation goes:** append it to the Description
under an `<h2>Technical Documentation</h2>` heading. Work-item rich text does
**not** render mermaid, so use numbered steps there. If the doc is long
(more than roughly two screens), keep it in the repo instead: commit
`docs/work-items/<slug>.md` (on a branch, never directly to `main`), put a
short summary plus a link to the file in the Description, and add the link as
a Hyperlink relation.

**d. Preview the exact commands**, then run them after confirmation:

```bash
az boards work-item create --type "User Story" \
  --title "<Title>" \
  --area "<Area path>" \
  --iteration "<Iteration path>" \
  --description "$(cat <tmpdir>/description.html)" \
  --fields "Microsoft.VSTS.Common.AcceptanceCriteria=$(cat <tmpdir>/ac.html)" \
           "Microsoft.VSTS.Scheduling.StoryPoints=<points>" \
           "<Reach ref>=<reach>" "<Impact ref>=<impact>" "<Confidence ref>=<confidence>" \
           "<RICE ref>=<rice>" "<Category ref>=<category>" \
           "System.Tags=<tag1>; <tag2>" \
  --query "{id:id, url:_links.html.href}" -o json

az boards work-item relation add --id <new-id> --relation-type parent --target-id <feature-id>
```

If a field value is rejected (for example, Confidence stored as `0.8` versus
`80`), read the allowed values from the error or the field definition, fix the
value, and retry. Don't drop the field silently.

**e. Report** the new work item ID and URL, which fields were set, and any
that the user still needs to fill in manually. Update the markdown file's
header with the new ID.

---

## Field reference (U of I standard)

A User Story represents a small, actionable piece of work that delivers value
to the end user, written in a user-centric format:
**"As a [user], I want [goal] so that [reason]."** Stories are broken down
into Tasks for implementation.

**Title.** A short, clear name describing the user story. Keep titles
consistent and descriptive.

**State.** The status of the story:

- **New**: Just created, not yet worked on. *(Default for new stories.)*
- **PO - Refinement**: Ready for review at a refinement session.
- **Development**: Work in progress.
- **Testing**: Work being tested.
- **Pending Release**: Ready for production release.
- **Closed**: Work is done and verified.
- **Removed**: Cancelled work.

**Area.** Specifies which product, service, or team owns the story
(e.g. `Development\Business Systems`).

**Iteration.** Assigns the story to a sprint or release cycle. By default, a
story is set to `Development` until it is added to the current sprint or
assigned to a future one.

**Description.** Definition of the problem, the business value, and any
assumptions or constraints. It opens with the "As a … I want … so that …"
sentence.

**Acceptance Criteria.** The conditions that must be met for the story to be
complete. They must be testable and measurable. See the standard below.

**R.I.C.E.** A prioritization score: **(Reach × Impact × Confidence) /
Story Points** for user stories.

**Story Points.** Effort estimate on the .5 Fibonacci scale (table below).

**Category.** Classifies the work by purpose:

- **Enhancement Request**: Improvements or modifications to existing features
  to enhance usability, performance, or efficiency.
- **Mandate/Risk**: Compliance, regulatory requirements, security risks, or
  mandatory changes that must be implemented.
- **New Functionality**: New features or capabilities that do not currently
  exist in the system.
- **Spike**: Time-boxed research or technical investigation to explore
  feasibility, technology choices, or unknown areas before development.
- **Technical Debt**: Refactoring, improving code quality, addressing
  inefficiencies, or resolving outdated dependencies.

**Related Work.**

- **Parent Link**: the Feature this story is part of. It must be a valid Feature.
- **Child Work Items**: Tasks and Bugs that further break down the story.
- **Dependencies**: use work-item links (Predecessor/Successor) when the story
  depends on another.

**Tags.** Used to categorize stories, filter in queries, and track related work.

**Prioritization context (MoSCoW).** When the user describes priority in
these terms, record it in the Description or Tags: *Must Have* (essential for
release), *Should Have* (important but not a blocker), *Could Have*
(nice-to-have), *Won't Have* (can be postponed).

### Hierarchy

Epic (large initiative) → **Feature** (major functionality within the epic) →
**User Story** (small, actionable development unit) → **Tasks** (specific
development, testing, or design activities). Every story belongs to a Feature.

### R.I.C.E. scale

| Reach | Impact | Confidence | Effort *(Epics/Features only)* |
|---|---|---|---|
| 2500 – Some of University | 3 – Massive Impact | 100% – High Confidence | 0.25 – 1 Day |
| 5000 – Most of University | 2 – High Impact | 80% – Medium Confidence | 0.5 – 3 Days |
| 10000 – All of University | 1 – Medium Impact | 50% – Low Confidence | 1 – 1 Week |
| 20000 – Beyond University | 0.5 – Low Impact | | 2 – 2 Weeks |
| 50000 – Mandated | 0.25 – Minimal Impact | | 3 – 3 Weeks |

For **User Stories**, the denominator is **Story Points**, not the Effort
column. Example: Reach 10000 × Impact 3 × Confidence 0.8 / 3 points = **8000**.

### Story Points (.5 Fibonacci)

| Points | Meaning | Typical use case |
|---|---|---|
| 0.5 | Very small task with minimal effort | Minor UI tweak, simple bug fix, small config change |
| 1 | Small task, straightforward with low complexity | Minor feature adjustment, adding a button, simple API call, database query update |
| 2 | Slightly larger task, still well understood | Small component development |
| 3 | Moderate complexity | New UI form, API integration, database changes |
| 5 | Significant effort, more unknowns, potential blockers | Full feature development, major refactor, performance improvements, general research spikes |
| 8 | Large and complex, may require breaking down | Multi-feature implementation, complex migrations, changes requiring multiple developers |

- 0.5 & 1 → quick wins, easy to estimate.
- 2 & 3 → small-to-moderate complexity, commonly used.
- 5 & 8 → larger work that might need refinement or splitting before a sprint.
  **At 8, propose a split.**

The examples are illustrative, not a hard rule. Use judgment based on the
story's context, and state your reasoning.

---

## Acceptance Criteria standard

### Characteristics of good acceptance criteria

- **Clear and unambiguous**: easy for developers, testers, POs, and
  stakeholders to understand. No jargon or vague language.
- **Testable / verifiable**: the most critical one. Each criterion can be
  definitively tested, manually or automatically. If you can't test it, it's not
  good acceptance criteria.
- **Concise**: get straight to the point, with no unnecessary detail.
- **Specific**: describe *what* the system should do, not *how*. No
  implementation details. Those belong in Technical Documentation.
- **Achievable**: realistic and within the scope of the story.
- **Independent**: ideally each criterion stands alone, though a natural flow
  is acceptable.
- **User-focused**: the desired outcome from the user's or stakeholder's perspective.

The structure deliberately mirrors **AAA (Arrange, Act, Assert)** from test
automation: GIVEN = Arrange, WHEN = Act, THEN = Assert. Because delivered
work is tested against the AC, write each scenario so it translates directly
into a test plan.

### Format (Gherkin)

```
SCENARIO 1: [Description of your first Acceptance Criteria]
  GIVEN [Initial context, setup, pre-conditions]
  WHEN [What happens to trigger this acceptance criteria]
  THEN [A verifiable expected outcome or response]

SCENARIO 2: [Description of your second Acceptance Criteria]
  GIVEN
  WHEN
  THEN
```

Use `AND` under any heading to add context, conditions, actions, or outcomes.
Each scenario should read naturally as an English sentence while stating
exactly what is being tested. Avoid unmeasurable words ("fast",
"user-friendly", "quickly"). Use numbers ("within 3 seconds") or observable
states ("the status shows *Paid*"). If the PO hasn't set a number, write
`<X — confirm with PO>` and list it under Open Questions.

### Example

```
SCENARIO 1: Successful Password Reset
  GIVEN I am a registered user and have forgotten my password.
  WHEN I navigate to the "Forgot Password" page and enter my registered email address.
    AND I click the "Reset Password" button.
  THEN I receive an email containing a unique, time-sensitive link to reset my password.
    AND Clicking the link directs me to a "Set New Password" page.
    AND After successfully setting a new password, I am able to log in with my new credentials.

SCENARIO 2: Invalid Email for Password Reset
  GIVEN I am on the "Forgot Password" page.
  WHEN I enter an email address that is not registered.
    AND I click the "Reset Password" button.
  THEN I see an error message "Email address not found."
    AND No password reset email is sent.
```

---

## Technical Documentation standard

The Technical Documentation section makes the story **hand-off ready**. A
developer who wasn't in this conversation should be able to carry out the
change from this section alone. It must be **complete enough to execute and
short enough to use**: use tables instead of prose wherever possible, don't
restate the Description or AC, and leave out any subsection that doesn't
apply. Never pad it with empty headings or boilerplate.

### Required structure

1. **Table of contents** at the top, with links to each included subsection.
2. Include the subsections below **that apply**, in this order.

**Overview & scope**: two to four sentences on the technical approach, plus
an **In scope / Out of scope** bullet list.

**Data model / schemas**: for every table, model, or schema that is created or
changed, include:

- A column table:

  | Column | Type | Nullable | Default | Allowed values / constraints | Description |
  |---|---|---|---|---|---|
  | `payment_id` | `UUID` | No | `gen_random_uuid()` | PK | Unique payment identifier |
  | `status` | `VARCHAR(20)` | No | `'pending'` | see data dictionary | Current processing state |

  Mark each column **(new)**, **(changed: was …)**, or leave it unmarked if
  unchanged but relevant.
- **Keys, indexes, and constraints**: PK, FKs (with the referenced table and
  on-delete behavior), unique constraints, and indexes, each with a one-line
  reason.
- **Relationships**: how this table relates to others (1:N, N:M via …).
- **Data dictionary**: every enumerated, coded, or non-obvious value, with its
  meaning:

  | Field | Value | Meaning |
  |---|---|---|
  | `status` | `pending` | Submitted, awaiting gateway response |
  | `status` | `paid` | Gateway confirmed, posted to ledger |
  | `status` | `failed` | Gateway declined, or retries exhausted |

- For non-relational schemas (JSON payloads, Pydantic/ORM models, event
  messages), use the same column-table format with `Field` instead of `Column`,
  and give nested objects their own table.

**Endpoints / APIs**: one block per endpoint:

| Item | Detail |
|---|---|
| Method & path | `POST /api/v1/payments/{id}/confirm` **(new)** |
| Auth | Role / scope required |
| Request | Path/query params and body schema (link to the schema table) |
| Response | Success status + body schema |
| Errors | Each status code → condition |

Then add one minimal request/response example in a fenced `json` block.

**Services / microservices / components**: for each service or module
touched: its responsibility, inputs and outputs, upstream and downstream
dependencies, and configuration (env vars and settings, with defaults and
which environment they are set in). Never include real secret values. Name
the variable and where it is stored (e.g. Key Vault, pipeline variable).

**Process flow**: numbered steps for the runtime flow (who calls what, in
what order, and what happens on failure). Add a mermaid `sequenceDiagram` in
the markdown file when there are three or more actors (remember it won't
render inside an ADO work item).

**Implementation steps**: an ordered, file-level checklist the next developer
can follow:

1. `backend/app/db/models/payment.py`: add `status` column (see schema).
2. `backend/alembic/versions/`: generate migration. Backfill existing rows to `paid`.
3. …

Reference real paths from the codebase. Mark new files **(new)**.

**Migrations, rollout & rollback**: migration order, backfills, feature
flags, deployment ordering between services, and exactly how to roll back.

**Testing notes**: map each AC scenario to the test(s) that prove it (unit,
integration, or manual), plus any test data or fixtures needed.

**Open questions**: every `TBD` from anywhere in the document, each with
the person or role who can answer it.

---

## Output template

Use this exact structure for the markdown document:

```markdown
# <Title>

| Field | Value |
|---|---|
| Work Item Type | User Story |
| ID | <ADO ID once created, else "Not yet created"> |
| State | New |
| Area | <Area path> |
| Iteration | <Iteration path — default Development> |
| Category | <Category> |
| Story Points | <points> — <one-line rationale> |
| Reach | <value – label> |
| Impact | <value – label> |
| Confidence | <value – label> |
| R.I.C.E. | <score> = (<reach> × <impact> × <confidence>) / <points> |
| Tags | <tags> |
| Parent Link | <Feature ID – Feature title, or "TBD — Feature needed"> |
| Child Work Items | <Task IDs/titles, or proposed tasks> |
| Dependencies | <predecessor/successor items, or "None"> |

## Description

As a <user>, I want <goal> so that <reason>.

<Business value, assumptions, constraints — a short paragraph or bullets.>

## Acceptance Criteria

SCENARIO 1: <…>
  GIVEN <…>
  WHEN <…>
  THEN <…>

SCENARIO 2: <…>
  …

## Technical Documentation

### Contents
- [Overview & scope](#overview--scope)
- [Data model / schemas](#data-model--schemas)
- …

### Overview & scope
…

(remaining applicable subsections, in the order defined above)
```

---

## Example

```markdown
# Instant Processing of Tuition Payments with Immediate Confirmation

| Field | Value |
|---|---|
| Work Item Type | User Story |
| ID | Not yet created |
| State | New |
| Area | Development\Business Systems |
| Iteration | Development |
| Category | Enhancement Request |
| Story Points | 3 — gateway API integration plus a status column; well understood |
| Reach | 10000 – All of University |
| Impact | 3 – Massive Impact |
| Confidence | 80% – Medium Confidence |
| R.I.C.E. | 8000 = (10000 × 3 × 0.8) / 3 |
| Tags | payments; student-accounts |
| Parent Link | XXX1 – Optimized Payment Processing |
| Child Work Items | XXX3 – Research and select the best payment gateways for low-latency processing |
| Dependencies | None |

## Description

As a student, I want my tuition payment to be processed instantly so that I
receive confirmation without delays. This lets me complete enrollment and
access university services without worrying about payment processing issues.

## Acceptance Criteria

SCENARIO 1: Successful payment is confirmed immediately
  GIVEN I am a logged-in student with an outstanding tuition invoice
  WHEN I submit a valid payment
  THEN the payment is processed within <X — confirm with PO> seconds
    AND I see an on-screen confirmation with a confirmation number
    AND I receive a confirmation by email and SMS
    AND the invoice status shows "Paid" in the university financial system

SCENARIO 2: Transient gateway failure is retried
  GIVEN I have submitted a valid payment
  WHEN the payment gateway returns a temporary error
  THEN the payment is retried automatically
    AND I see a "processing" status rather than an error while retries run

SCENARIO 3: Payment ultimately fails
  GIVEN I have submitted a payment
  WHEN the payment is declined or all retries fail
  THEN I see a clear error message explaining the payment did not go through
    AND the invoice remains "Outstanding"
    AND no confirmation email or SMS is sent

SCENARIO 4: Processing time is observable
  GIVEN payments are being processed
  WHEN a staff member opens the payments dashboard
  THEN they can see processing time per transaction
    AND transactions exceeding <X> seconds are flagged as delayed

## Technical Documentation
… (Contents, Overview & scope, Data model with the `payments` column table and
`status` data dictionary, Endpoints, Process flow, Implementation steps,
Rollout & rollback, Testing notes, Open questions: "Target processing time X —
PO")
```
