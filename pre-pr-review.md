---
name: pre-pr-review
description: |
Self-review checklist that MUST run before opening, updating or pushing any pull request
(iute-processes back end or iute-back-office front end). Catches the classes of mistakes
reviewers found on APP-528 (#273, #274, #45, #46, #47) and APP-546 (#284): production-safety gaps,
provider limits taken from our code instead of the provider spec, half-handled known issues,
hidden-field validation, missing MVVM-layer tests, ignoring the repo's test/helper patterns,
convention violations a rule-by-rule pass would catch (HTTP exceptions in services, clever
expressions, extra comments), behaviour broken by a change in a neighbouring task, and migration
numbers already taken by another open PR; and the UI behaviour gaps reviewers found on #46–#48
(loading states, tab switches, typing, seeded data, list limits, logic in views, partial tests,
duplicated helpers left behind); and on APP-569 (#293): new code duplicating an existing
function instead of reusing it, repo CLAUDE.md rules skipped, policy examples applied without
checking their condition, and PR-level comments left unread.
user-invocable: true
when_to_use: |
Right before `gh pr create`, before pushing new commits to an open PR, and before saying
"ready for PR" / committing a feature the user will turn into a PR. Also when the user asks
for a self-review of a branch.
allowed-tools:
- Read
- Bash
- Grep
- Glob
---

# Pre-PR self-review

Run this on the branch diff against its base (`git diff <base>...HEAD`) before every PR, before
every push to an open PR, and after every rebase or rewrite of an already reviewed branch.
Go through every section. For each item, either fix the problem, or write down why it does not
apply. Put anything noticed but deliberately not fixed in the PR description under
**Known limitations**. Never leave it silent.

**This is a search for defects, not a confirmation.** The goal is to find what is wrong.
- Read the full diff file by file, line by line. A section is done only when every changed line
was checked against it.
- Never write "no findings", "compliant" or a list of rules that were followed without having
walked every rule over every changed file. "Rules followed: CC#1, #2…" is not a review.
- Treat your own code as someone else's: assume it has bugs and look for them.
- **Fix in-scope findings immediately**, then re-run the gates. "Known limitations" is only for what
needs another repo/PR, a user decision, or is pre-existing outside the ticket. "Not my task" is not
a reason when the code is in this branch or the line was touched by it.
- **Read every review thread first.** Before changing or pushing any PR — including the one you are
rebasing as a side effect — list its unresolved threads and "changes requested" reviews
(`gh api repos/<owner>/<repo>/pulls/<n>/comments`, `.../reviews`) **and the PR-level comments**
(`gh api repos/<owner>/<repo>/issues/<n>/comments`, where review bots and some reviewers post).
Never report a PR as ready while it has comments you have not read.
- Review the **whole PR**, not only the latest commit, and every PR the branch is built on that is
still open (stacked PRs) when the user asks to review a story. **On every push to an open PR, walk
the whole PR diff against its base again** (`git diff <base>...HEAD`), not only the delta since the
last push: earlier commits keep the defects the previous pass missed. Example (#378): six comments
of a second reviewer were in the first commit, which later passes never re-read.
- **A review comment names one place, the fix covers all of them.** After fixing a comment, grep the
whole PR (and the code it touched) for the same pattern and fix every occurrence in the same push.
Example (#378): `error` → `warn` was fixed for the two lookups the reviewer pointed at; the same
refusal in template delete stayed `error` and came back as a new comment.

Report the result to the user as findings: `file:line` → rule or section → what is wrong → what was
fixed (or why it stays, and where it is disclosed). Lint, format, type-check and test counts go
last (section 8); they are never the headline of the review.

- **Evidence, not volume.** Each finding names the trigger (concrete input or state), the impact and
a verified `file:line`; never invent paths or lines. Do not split one root cause into several
findings, and do not add findings to fill a quota.
- **Separate pre-existing problems** from ones this branch introduces or touches; report the former
as such.
- **When nothing is found**, say so together with the scope that was reviewed and what was not
verified (e.g. e2e not run on this branch). "No findings" is never a claim that the code is safe.

## 1. Production safety: can this action break delivery or access?

- For every mutation, toggle, delete or reorder, ask: **after this action, does the flow still work
end to end?** For example: is a notification still deliverable, can a customer still log in or
receive an OTP, does a workflow still have a path.
- Look for operations that can leave a critical config empty or fully inactive. Examples: switching
off the last active provider of a key, saving a cascade where every provider is inactive,
removing the only template.
- Either block them on the back end with a clear error,
- or return the impact (e.g. which templates are affected) so the front end can show it and ask
for an explicit confirmation.
- Check money, auth and OTP paths first. They are the costliest to break.
- **Write down the assumptions the feature relies on** (e.g. "every term has a default-language
text", "the bundle contains every key", "the default language is uk") and check each one against
real data, not against data the tests create. If a neighbouring task in the same story changed
one of them during the work, re-check everything in this branch that relies on it.
- Transitional code (dual writes, fallbacks) must stay correct in every language and after edits
made elsewhere; if the release ships the dependent parts together, prefer no transitional code.
- **Concurrent edits.** For every update of state a person edits in a form, ask: can a save made
from an outdated form silently overwrite someone else's change? Require a token from the client
(`version` / `updatedAt` of the state it was edited from) and check it in the same
`UPDATE ... WHERE`. A lock on a value the server reads itself does not protect against a stale
form. Example (#361): a settings-only save did not bump the version, so a parallel content save
from an older form passed the check and wiped the new BCC.

## 1b. Security review: try to break it, not only to use it

Functional and regression checks prove the happy path. For every change to sign-in, authorization,
secrets or infrastructure, also act as an attacker and as an operator, and run the attempts — do not
reason about them.

- **Walk the security policy rule by rule** (`security.policy.md`, every numbered rule) over every
changed file, the same way as code-convention: a finding with `file:line`, or why it does not apply.
Infra scripts, workflows and realm config are in scope. Example (#393): brute-force protection off
broke §8, a printed secret and a year-long old secret broke §9 and §12, one app for every environment
broke §3 (least privilege).
- **Bypass attempts for every sign-in change.** List every way to reach a password or a token and try
each on a running instance: empty or other IdP hints (`kc_idp_hint=`), `prompt=login`, every client of
the realm with direct grants (also built-in ones like `admin-cli`), the account console, the admin
API. "The redirect works" is not "the password is off". Example (#393): with `kc_idp_hint=` the
Keycloak form opened and the repo test password signed in.
- **Brute force and rate limits** on every path that checks a password or a code (§8).
- **Access lifecycle.** When a person leaves or loses a role: how long do existing sessions, refresh
tokens and cached roles keep working, and what does an operator have to do? Session lifetimes kept
from an earlier decision are not automatically right for a new sign-in method. Example (#393): a
week-long session with roles synced only at login.
- **Secret lifecycle.** Where a secret appears (terminal, logs, CI output, files), where it is stored,
how it is rotated and when the old one is revoked.
- **Blast radius.** One credential, app or role per environment; a leak or a role in dev must not reach
stage or prod.
- **Challenge the spec.** A ticket or plan is a design to review, not a proof of safety: check it for
least privilege and blast radius before implementing it literally.
- **Never park a security finding** under Known limitations on your own: fix it, or ask the user. Do not
argue it away ("the hole already exists elsewhere") without checking that the other path really stays.

## 2. External limits: take them from the provider spec, not from our code

- Every value sent to an external provider (Viber/GMSU, SMS, push, Diia, SR Bank) is validated
against **that provider's documented limits**. Examples: Viber `ttl` 40–86400 seconds, button
caption ≤ 30 characters, message length, URL format.
- Defaults or usages in our code (e.g. `GMSU_VIBER_DEFAULT_TTL = 60`) are not the limits. Find the
spec (vendor docs in RAGFlow "Vendor documentation", the provider client, or ask the user).
- Enforce the same limits on **both** the back end (DTO/class-validator) and the front end (Zod),
with clear messages.

## 3. Known issues: finish them or disclose them

- A limitation you can fix in this ticket is a finding, not a "Known limitation". Example (#378):
orphan webhook events kept forever were listed as a limitation; the reviewer asked for the cleanup.

- Re-read your own earlier analysis, notes and screenshots in this task. Anything you flagged
(e.g. "groups disappear in collapsed sidebar", "the add-list shows options the back end will
reject") is either fixed in this PR or listed under **Known limitations** in the PR body.
- Do not offer options in the UI that the back end will reject. Filter them on the front end
(e.g. only providers whose channel has a template for the key).
- If a mutation returns the new state, use it directly. Do not refetch and flash stale data.
- **Check your own fixes from the other side.** When a finding is fixed, walk the same scenario
again with the roles swapped (e.g. settings-only save vs content save, first save vs second,
create vs update). A fix that closes one order of events and leaves the other open is a finding.

## 3b. Incoming data, logs and stored data

- **Incoming external payloads** (webhooks, callbacks, partner responses): for every field the code
reads, ask what happens when it is missing, empty, of another type or too long. Drop or reject the
item; never default it to a sentinel (`0` → a 1970 date, `''`). Example (#378): a SendGrid event
without `timestamp` was stored as 1 January 1970 and could win the ordering.
- **Log levels and volume.** A refusal caused by the caller (wrong ID, forbidden action, a template
that cannot be deleted) is `warn`; `error` is for failures we must fix. A condition that repeats on
every request (missing config, a provider retrying for a day) is logged once (at start), not per
request.
- **One source for stored data.** Do not store a value that is never read, or store it and read that
copy; never keep two versions of the same fact (stored and re-derived) where only one is shown.
Example (#378): the snapshot stored the subject, the back office showed a re-rendered one.
- **Queries added by the change.** For every new DB round trip, check whether an existing statement
already returns the value (`.returning`, the row just read). Example (#378): an extra `SELECT` only
to read a status the `UPDATE ... RETURNING` could return.

## 4. Forms: what the user cannot see must not block them

- Fields hidden by a condition (channel, toggle, tab) are either:
- validated only when visible (conditional Zod / `superRefine`),
- or cleared when the condition changes.
- Walk the scenario: fill a hidden-able section with invalid data → switch the condition → submit.
Save must work or show a visible error.
- Errors in a non-active tab must be visible: mark the tab, or switch to it.
- Toasts behind an open modal's overlay are invisible. Show the confirmation inside the modal.

## 4b. Behaviour walk: a scenario table for every changed UI piece

Checking rules against the diff does not find behaviour bugs. For **every changed screen, dialog,
tab, form field and list** write a table in the report — one row per scenario, with what happens
in the code (cite `file:line`) — and fix every row whose answer is wrong:

| Scenario | Ask |
|---|---|
| Typing | Type into each text input character by character, including spaces at the start, middle and end. Does a schema/`trim` run on every keystroke and eat input? Trim once, on the debounced or submitted value. |
| Loading | While each query is still fetching (first load and refetch), what do buttons, validations and messages say? Is every query's `fetching` and `error` part of the state that gates actions (e.g. no "no active provider" while templates load)? |
| Errors | Each query/mutation fails: is there a visible message and a retry? |
| Tabs / conditions | Change something, switch tab or condition, come back. Radix `Tabs` unmount inactive content: is the draft kept (lift state or `forceMount`)? Are errors in a hidden tab visible? |
| Real data | Open the records the database really has (seeds, migrated rows: null titles, one-language texts, other channels) — query the local DB. Can each be opened, edited and saved? Is a field required only where the channel/type needs it? |
| Limits | More rows than the page/limit the UI loads (`limit: 100`, `20`): is it visible that the list is cut? Empty list? |
| Permissions | Without READ / WRITE: hidden or disabled controls, no queries fired. |
| After save | The UI shows the returned state at once, without flashing old data. |

Also, for every changed view/component: any value **computed** in JSX beyond plain formatting
(flags like `active`, `busy`, filtering, key building) moves to the viewmodel (CC#3). Every i18n key
a module uses lives under that module's namespace or in a shared/common namespace — never another
module's keys.

## 5. Tests: match the repo's layers and helpers

- **Front end (policy #9: test each MVVM layer).** Every module has:
- `__tests__/{module}.scheme.test.ts` (trim, max lengths, unknown enum values);
- `{module}.viewmodel.test.ts`;
- `{screen}.view.test.tsx` with a mocked viewmodel (error + retry, skeleton, empty list, disabled
controls without permission);
- an integration test.
Mirror the sibling module (e.g. `notification-templates`).
- **Back-end e2e.** Use the generated SDK from `@iute/test-utils` (`backOfficeSdk`, `sdk`) like the
newest specs do (`localization.e2e.spec.ts`). Do not write raw GraphQL strings. If an operation
is missing from the SDK, add it to the SDK documents and regenerate.
- **Before writing a test, open the two most recently changed specs** in the same app and copy
their structure and helpers.
- Cover the safety cases from section 1 and the limits from section 2 with tests.
- **Coverage per function, not per file.** List every function/handler the viewmodel returns and
every branch of a changed component; each needs a test (save, reset, error paths, status variants
like DRAFT, permissions). A file that exists but tests one function is a finding. New private
components (tabs, dialogs) get a view test with a mocked viewmodel.
- **Mocks must not bypass the boundary at risk.** If a test mocks the repository, client or
resolver whose behaviour the change is about, it proves nothing about that behaviour; cover it
with the e2e/integration layer instead.
- **Every acceptance criterion and spec field has an assertion.** Map each AC / spec bullet the PR
implements to the test line that checks it (e.g. audit `actor`, `customerId`, message ID). A field
that is set but not asserted is a finding. Example (#378): the audit actor was written but no test
checked it.
- **Test on the data the system really has.** Besides data the test creates, include the shape a
migration or production produces (e.g. a term migrated with only the Ukrainian text while the
default language is en). Tests that only build data matching the implementation's assumptions
prove nothing.

## 6. Code shape and conventions

- **One name per concept.** No re-export aliases (`export { X as Y }`). Import the shared name
everywhere and delete the alias file.
- **Reuse before writing.** For every new function, query, helper, hook or component in the diff,
search the repo for code that already does the same job (same tables/columns, same filter, same
formatting, same UI pattern), e.g. `grep -rn '<table or column>\|<key word>' apps/*/src src`.
Check the module that owns the concept first (customer search lives in the customer module).
- If it exists: reuse it, or extract one shared function into the owning module and switch
**both** callers to it. Do not write a second implementation next to the first.
- Do not widen a generic helper (e.g. `buildRelatedWhere`) to cover a case a domain function
should own; extend the domain function instead.
- Record in the report, per new function, what was searched and what was found.
Example (#293): onboarding search by name/tax ID/phone was written next to the same search in
`customer.service`; the reviewer asked for one function in the customer module used by both.
- **Reuse covers libraries and your own new code.** Before writing a validator, parser or header
builder, check the dependencies already in `package.json` (`class-validator` `isUUID`, …). Compare
the new call sites of this PR with each other: two places that build the same object (e.g. the
SendGrid and the fake strategy building the email snapshot) get one helper taking the smallest
input (`input`, `content`), so callers cannot pass inconsistent fields. Copying an existing block
(e.g. `Content-Disposition` from another controller) is a finding: extract it and check it against
the standard (RFC 5987 for `filename*`).
- **Fixed value sets in the API are enums.** A GraphQL field with a known set of values is a
`registerEnumType` enum, not `String` (CC#4 applies to the API as well).
- **No leftover copies.** When the branch introduces or uses a shared helper (e.g. `ilikeContains`),
grep the repo for the old inline copies (`grep -rn '<pattern>' src`) and replace them all.
- **Exports and guards.** Do not export what nothing imports; drop guards for values the types say
cannot be empty (e.g. `!template.templateKey` on a non-nullable field).
- **Readable transactions.** Split long `db.transaction` callbacks into named steps (validate →
load → write). Prefer the repository API (`find`, `findOne`, `create`) where it is enough; drop
to the raw query builder only when needed.
- **Read cost of list fields.** For every field a list or page query returns, ask: does the list
load heavy data (HTML, blobs, large json) for every row even when the client does not ask for
it? Heavy or optional nested data goes to a `@ResolveField` batched per request with a
DataLoader — never a query per row (CC#11). Example (#361): every template in a 100-row list
loaded its full email HTML although the message filter only needed `templateKey`.
- **One source of config.** Do not copy another service's env values or defaults into this one.
Store `null` and resolve the default where its config lives. Check mechanically: grep every new
env name over `k8s/` and the `.env.local` files; the same name in two services is a finding.
Example (#361): `SENDGRID_FROM_EMAIL` was copied into back-office and already differed
(`noreply@` vs `no-reply@`).
- **New tables follow the repo.** Take the id type from the majority of existing tables
(`git grep -n "uuid('id'\|bigint('id'" -- 'apps/*/src/**/*.schema.ts'`, uuid7 by default); a
column whose DTO has a length limit gets the same length (`varchar(n)`), not `text`.
- **Readable expressions.** An object literal with several conditional spreads
(`...(x && {...})`), a ternary inside it, `...(x ? [..] : [])`, chains of `flatMap`/`map` with
nested lambdas and ternaries, or an IIFE that calls anything (CC#37) are findings. Fix with named
builder methods (`toXRequest`, `senderOf`, `errorsOf`), plain `if` + `push`, and loops whose body
is extracted when it has nested blocks (CC#7). A repo `CLAUDE.md` rule against single-use
methods covers branches and guards, not builders the reviewer has to read.
- **Rule-by-rule policy pass.** Load the policy skills (`code-convention-backend` /
`code-convention`) and, for every changed file, walk **every numbered rule** of the code-convention
policy (back end 1–36, front end 1–11) and the architecture file-responsibility table. Record per
file either a finding with `file:line` or that the rule does not apply. Typical misses:
- services hold business logic only: no `HttpException`/`BadRequestException`/GraphQL concerns
in a service — throw domain errors and map them to transport errors at the resolver/filter;
- plain early returns and `if`s instead of clever expressions (`[a && X, b && Y].find(Boolean)`);
- types and DTO/entity style: when a policy example and the surrounding repo code differ (e.g.
`field?: string | undefined` vs `field?: string` on an entity), follow the surrounding code or
ask — do not apply the example mechanically;
- **rules with a condition**: before applying a rule or example that depends on a setting, check
the setting in the repo config (`tsconfig.json`, `biome.json`, `.oxlintrc.json`). Example (#293):
CC#18 `?: T | undefined` applies only with `exactOptionalPropertyTypes: true`; the repo has
`false`, so its CLAUDE.md rule "`?:`, never `| undefined`" wins. Check mechanically:
`git diff <base> -- ':!**/*generated*/**' | grep -nE '^\+.*\?:.*\| undefined'`;
- copying a pattern from the existing file does not make it compliant; check it against the
policy too.
- **Repo CLAUDE.md pass.** Read the repository's `CLAUDE.md` (and the `CLAUDE.md` of every app
folder the diff touches) and walk each of its rules over every changed file, the same way as the
numbered policy rules. Review bots check against it; it is not covered by the RAGFlow policies.
- **Comments: match the surrounding code's density.** No JSDoc blocks, migration headers or
explanatory comments unless the neighbouring code has them. Explain decisions in the PR body.
Check mechanically: `git diff <base> | grep -E '^\+.*(//|/\*|#\s)'` (and the untracked files) must
show no comment you added. A comment you think is "needed" still goes to the PR body instead.

## 7. Scope and PR shape

- **One feature = one branch = one PR into the story**, unless the user says otherwise. Include
only the tickets the user asked for.
- If a feature depends on another unmerged branch:
- say so in the PR body;
- **tell the user before opening the PR** when the dependency is something they asked to
exclude (e.g. a front end that needs a back end you were told not to include).
- Check that the base branch is fresh (`git fetch`) and the PR is MERGEABLE after opening.
- **Migrations:** before opening the PR, list the open PRs that add files to the same migrations
folder (`gh pr list` + `gh pr view <n> --json files`) and make sure the migration numbers do not
collide. If they do, tell the user and agree who renumbers.
- **Decisions:** re-read the user's decisions in this task and check the code follows what was
decided, not what you assumed (e.g. "these ship together" means transitional code is not needed).

## 8. Final gate

- Lint, format, type-check and unit tests pass. Relevant e2e/Cypress specs pass against a locally
running build of **this branch**: restart or clean-rebuild containers after branch switches or
rebases.
- The PR body is in English and lists: Why, What, Tests, Known limitations (if any), merge order
and dependencies.

