# GitLab Source for dsh-superagent — Design

Status: draft for operator approval. Date: 2026-11-19. Branch: `gitlab-source`.

## Intent

The operator runs dsh-superagent on a machine whose team works on a self-hosted GitLab
(`gitlab.xiding.tech`, GitLab CE 19.5). The prototype supports only GitHub and Feishu Sources,
so dsh-superagent cannot be installed as-is. The operator asked to install it on GitLab and
chose the scope: **Issue creation, closing and comments; MR creation, review and comments, and
merge**. CI pipeline events are explicitly out of scope for now.

The purpose is not "GitLab support" in the abstract: it is one working installation whose
agent takes a request from a GitLab Issue/MR comment, works in its own DSH Session and
checkout, answers in the same thread, opens a Merge Request, answers review comments, and
ends the Ticket when the MR merges or closes.

## Goals

1. A new Source kind `gitlab` that maps GitLab events to the existing domain (Trigger,
   Thread, Eval, Ticket, Mandate, Channel) without changing core semantics.
2. The events in scope:
   - Issue created (observed switch, like GitHub's `observeIssues`);
   - Issue closed/reopened as a Thread event delivered to subscribed Tickets;
   - Issue comment (mention-addressed, or any comment with `observeComments`);
   - MR created (observed switch, like `observePulls`);
   - MR review: approval and unapproval, plus review/diff discussion comments;
   - MR merged/closed: ends the bound Ticket (same lifecycle as a GitHub PR);
   - MR comment, including replies inside a diff discussion.
3. A GitLab agent identity through a `glab` wrapper, isolated exactly like the existing
   `gh` wrapper: worker shells and host CLI calls act only as the agent account, git uses that
   account's credential helper, and the operator's own tokens never leak in.
4. Deployment/setup support: resolve the GitLab project and agent account from settings,
   verify push access and the CLI login, write the worker environment, report structured
   setup problems in the panel.
5. Keep the GitHub Source working and tested: the bundle's default behaviour is unchanged.
6. Install and configure the running instance on this machine against the operator's GitLab,
   and verify it end to end.

## Non-goals

- CI pipeline events (deferred by the operator; a later Source change, not this one).
- GitLab webhooks (polling matches the existing architecture and survives restarts; an inbound
  HTTP listener needs reachability the operator's host does not have).
- Multiple GitLab projects per Source instance; group-level Sources; epics/boards.
- GitLab notifications inbox / todos as a separate stream (mentions are caught from the
  updated-issue/MR note reads; no todo state is written).
- Images in notes (`/uploads`), GitLab Pages, Wiki, snippets, submodules.
- Removing or renaming the GitHub Source; a "forge abstraction" refactor.
- Any change to how Feishu works.
- Changing the one-repository-per-installation assumption.

## Anti-goals

- A half-working Source that creates Tickets but cannot reply, bind an MR or end a Ticket.
- Prompt text that tells the agent to use `gh`/pull requests in a GitLab-only installation.
- Regressions in the GitHub Source, its prompts, or its tests.
- New secrets in configuration files, logs, prompts or the database. The wrapper holds the
  account's credentials in the operating system keyring, as `gh-agent` does today.
- Duplicate comments or duplicate Tickets after a restart: every write and every event must be
  idempotent by construction.

## Constraints and verified facts

Environment probed on this machine (2026-11-19):

- Node 24.22.0, pnpm 11.7.0, git 2.39.5, tmux 3.8, Python 3.9.6, `gh` 2.102.1 (signed in as
  `eanfs`), `glab` 1.124.0 (signed in to `gitlab.xiding.tech` as `lirichen`), no `lark-cli`.
- DSH 0.2.2-rc.1 installed; `package.json` requires `engines.dsh ^0.2.1-alpha.2`.
- GitLab instance is self-hosted CE, API at `https://gitlab.xiding.tech/api/v4`.
  Read-only checks confirmed these endpoints and shapes on an accessible project:
  - `GET /user` returns `id`, `username`, `name`, `email`, `commit_email`;
  - `GET /projects/:id` returns `path_with_namespace`, `web_url`, `default_branch`,
    `permissions.project_access.access_level` / `group_access`;
  - `GET /projects/:id/issues?updated_after=<iso>&order_by=updated_at&sort=asc` and the same
    for `merge_requests`;
  - `GET .../issues/:iid/notes`, `GET .../merge_requests/:iid/notes` (system notes and
    diff notes included, `author`, `created_at`, `system`, `type`, `position`,
    `discussion_id`, `in_reply_to_id`);
  - `GET .../merge_requests/:iid/discussions`;
  - `GET .../merge_requests/:iid/approvals` returns `approved`, `approved_by[]`,
    `user_can_approve`;
  - `glab api --paginate --output json <path>` returns one JSON array; `--hostname` selects the
    self-hosted instance.
- Repo conventions that must be respected: zod loose-object schemas validated before use; one
  `SourceKind` per provider; `Commands` is the only CLI transport and must stay
  repository-scoped; every module listed in the tables of `docs/007-architecture.md`
  (`tests/docs.spec.ts` enforces it); every class has a JSDoc block
  (`tests/class-docs.spec.ts`); prompt text rules are asserted by `tests/cards.spec.ts`;
  prompts/skills are size-budgeted by `tests/model-surface.spec.ts`.

## Approaches considered

**A. A parallel `gitlab` Source kind (chosen).** New source modules structured like the
GitHub ones (`gitlab-trigger`, `gitlab-channel`, `gitlab-read`, `gitlab-mr`, `gitlab`), a
`glab` transport in `Commands`, optional `gitlab` block on `Deployment`, `glab` shim and
GitLab git-credential in the worker environment, a built-in `skill/gitlab`, source-neutral prompt
wording, resources/panel/docs/tests additions. Keeps GitHub intact; every piece is small and
testable. Cost: broad but shallow change (~45 files, 2k+ lines including tests and docs).

**B. Refactor into a shared "forge" layer.** Extract the GitHub Source into a provider-neutral
core with GitHub/GitLab adapters. Cleanest long-term, but rewrites the prompt, skill, resource
and panel layers and risks the one thing that currently works; the prototype would not pay for
it. Rejected.

**C. Minimal side plugin with install-time prompt overrides.** A standalone `gitlab-watch`
plugin that emits Triggers, leaving default prompts untouched and patching GitHub wording
through team prompt overrides at install time. Smallest code, but default prompts keep telling
the agent `gh`/PR, which makes behaviour flaky and the installation unreproducible. Rejected.

## Design

### 1. Identity, deployment and setup

`Deployment` (src/deployment.ts) gains an optional `gitlab` block and `github` becomes
optional, because an installation has one forge account, one repository.

```ts
interface GitlabAccount {
  cli: string          // wrapper script name or absolute path, e.g. ~/.local/bin/glab-agent
  host: string         // gitlab.xiding.tech
  login: string        // agent username
  userId: number       // agent user id (commit trailer)
  commitEmail: string  // /user commit_email
  project: string      // path_with_namespace, canonical
}
interface Deployment extends Machine {
  repository: string            // GitHub owner/name or GitLab group/project
  github?: { cli: string; login: string; noreply: string }
  gitlab?: GitlabAccount
  feishu?: FeishuBot
}
```

- `SETUP_FIELDS` gains `gitlab: ['project', 'cli', 'host']`; a change to one restarts the
  service so setup re-resolves the account.
- New `resolveGitlabDeployment(machine, config, probe, path)`:
  1. require `project` and `cli` (SetupError codes below);
  2. resolve the CLI executable, run `<cli> api --hostname <host> user`;
  3. run `<cli> api --hostname <host> projects/<urlencoded project>`;
  4. canonical `path_with_namespace` becomes `deployment.repository`;
  5. require push: `permissions.project_access.access_level >= 30` or
     `permissions.group_access.access_level >= 30` (Developer and above).
- Resolution order in `Setup.run`: if the GitLab source config names a project, resolve GitLab
  as the primary forge; else resolve GitHub as today. Both configured is refused with a new
  `repositoryAmbiguous` problem (one installation, one work repository). Neither yields the
  existing `repositoryUnset` problem.
- New `SetupPlace` value `gitlab` and codes: `projectUnset`, `gitlabCliUnset`,
  `gitlabCliMissing`, `gitlabCliSignedOut`, `gitlabCliNoAccount`, `projectUnreadable`,
  `projectUnreported`, `projectReadOnly`, `repositoryAmbiguous`; GitHub codes keep their text.
- `Setup.checkAccounts` maps `source:gitlab` to the GitLab resolver; `app/setup.ts` writes
  the environment for whichever forge resolved.

### 2. Transport: `Commands`

Add to `src/infra/commands.ts`, without changing the GitHub methods:

- `glab<T>(args)` — runs `deployment.gitlab.cli` and parses JSON;
- `gitlab<T>(schema, path, ...flags)` / `gitlabApi(path, ...flags)` — path is relative to
  `projects/<urlencoded project>/`; absolute paths, schemes and `..` are refused with the same
  rule as the GitHub transport;
- `gitlabPages<T>(item, path, tally?)` — one `glab api --paginate --output json` call, then
  `validItems` over the array;
- every glab call passes `--hostname <deployment.gitlab.host>`;
- errors: `commandError` learns glab's diagnostics (HTTP status in `glab: 404 Not Found` or a
  JSON `message` such as `404 Project Not Found`), so `CliError.code` carries the HTTP status
  as it does for `gh`. Non-HTTP failures stay retryable.

### 3. Source kind `gitlab`

`src/sources/gitlab.ts` exports the kind and the plugin entry (`name =
'dsh-superagent-gitlab'`, built as `lib/gitlab-watch.js`).

Configuration (zod, strict):

| field | type | default | meaning |
| --- | --- | --- | --- |
| `project` | text | — | `group/project` |
| `host` | text | `gitlab.com` | instance host |
| `cli` | text | — | wrapper path or command |
| `pollSeconds` | number 60–3600 | 60 | poll interval |
| `failuresBeforeError` | number 1–20 | 3 | consecutive failed polls before `error` |
| `observeIssues` | switch | false | receive new unaddressed Issues |
| `observeMerges` | switch | false | receive new unaddressed MRs |
| `observeComments` | switch | false | receive any new unaddressed comment |

- `health: ['gitlab']`.
- `scope`: `listened: true`; `observed` true only for the matching switch; row `issues`,
  `merges` or `comments` derived from Trigger `meta` exactly as GitHub's `scopeOf` does.
- `policy`: `allowed` usernames (`[A-Za-z0-9_.-]+`), optional `bots`; `explicit` text:
  an allowed account mentions the agent account; rows `issues`, `merges`, `comments` with the
  same three cells and defaults as GitHub.
- `environment.variables`: `DSH_SUPERAGENT_GITLAB_PROJECT`, `_HOST`, `_ACCOUNT`,
  `_COMMIT_EMAIL`, `_KIND` (`issue` | `mr`), `_IID`, `_SOURCE_BRANCH` (MR source branch,
  empty otherwise). Values come from the deployment and the originating Trigger's `meta`.
- `facts`: current account, as GitHub's card shows it.

### 4. Events → Triggers

All ids are derived with `key` and are stable across polls and restarts. Threads are
`gitlab:issue:<iid>` and `gitlab:mr:<iid>` (GitLab Issue and MR iids are separate number
spaces). Channel addresses carry `{ kind, iid, discussionId?, noteId?, url }`.

| GitLab event | Trigger id | Addressed | Notes |
| --- | --- | --- | --- |
| Issue created | `key('gitlab','issue',iid)` | mention in description | only when created after the stream started; never the agent's own |
| Issue closed/reopened | `key('gitlab','issue-state',iid,state,updated_at)` | none | delivered as Thread context to subscribed Tickets; never ends a Ticket |
| Issue comment | `key('gitlab','note',note.id)` | mention in body | system notes never become Triggers |
| MR created | `key('gitlab','mr',iid)` | mention in description | as "pull" is to GitHub; description may bind a Ticket |
| MR comment / diff comment | `key('gitlab','note',note.id)` | mention in body | `meta` carries `path`, `line`, `discussionId`, `replyTo` |
| MR approved / unapproved | `key('gitlab','approval',iid,revision)` | none | delivered to subscribed Tickets only; revision counts changes |
| MR merged/closed | (no Trigger) | — | observed state ends the bound Ticket with `threadClosed` |
| MR reopened | `key('gitlab','mr-state',iid,state,updated_at)` | none | context to subscribed Tickets |

Author rules: `human = author.bot !== true`; the agent's own notes/authors are dropped; bot
notes may only become inferred candidates (Policy `bots`), matching GitHub. Trigger `text`
contains the note/description, the actor username, the timestamp, and the `web_url`.

`find(query)`: `#12` → `gitlab:issue:12`, `!12` → `gitlab:mr:12`; also matches full URLs.

### 5. Polling model and cursors (KV)

New `KV.gitlab` family:

- `streams(kind)` for `issues` and `merges`: `{ since }`. The first poll sets `since` to the
  round start, so history is not read; later polls move it to the newest `updated_at`
  (`cursorAt`, 5 s back) after every item has been received.
- `cursor(threadKey)`: the last note timestamp processed for a Thread; advanced only after all
  its notes are received.
- `state(threadKey)`: last observed `state` (`opened`/`closed` for Issues;
  `opened`/`closed`/`merged`/`locked` for MRs) used to detect transitions.
- `approvals(iid)`: `{ fingerprint, revision }` like the GitHub CI fingerprint.
- `labels` is not needed; GitLab notes carry author names.

One poll round (abortable between items; a failure never advances a cursor):

1. **Streams (always on).** For Issues and MRs, `updated_after = since`, ascending, pages of
   100. For each item: receive the created Trigger when it is fresh and not the agent's;
   compare `state` with `state(threadKey)` and receive a state Trigger on change; read new
   notes (below). Then move the stream cursor.
2. **New notes.** `notesSince(commands, kind, iid, since)`: fetch notes newest-first in pages
   until a note is older than `since` (bounded at 10 pages = 1000 notes); return them oldest
   first. Skip the agent's own, skip system notes, skip unaddressed notes unless
   `observeComments` is on or the Thread already has subscriptions (subscribed Threads are
   where unaddressed discussion must reach the worker). Receive addressed notes always.
3. **Approvals.** For each polled MR, when it has a subscribed Thread or its state changed,
   read `/approvals`, compare the sorted `approved_by[].username` fingerprint; on change
   increment `revision` and receive an approval Trigger carrying the approvers and `approved`.
4. **Subscribed Threads** (`pollThreads` from `src/sources/poll.ts`): read state and notes since
   the per-thread cursor for every `gitlab:*` Thread with subscriptions; MR merge/close →
   `observeMerge`; Issues closed → state Trigger. This makes comments arrive to a working
   Ticket even with all observe switches off.
5. **Ticket MRs.** For open Tickets that follow `gitlab:mr:*` Threads, read the MR and call
   `observeMerge` (bind, and `threadClosed` on merge/close) even if the MR was not in the
   update window — the same reconciliation GitHub does for pull requests.
6. Report `ready` with counts, or `error` after `failuresBeforeError` consecutive failed rounds.

Mentions of the agent anywhere in the project are caught by step 1–2 without an inbox stream,
because every note changes its Issue/MR's `updated_at`. This replaces GitHub's notifications
stream; it costs one notes read per changed Issue/MR per minute, which one project tolerates.

### 6. Channel and thread context

- `src/sources/gitlab-channel.ts`: `sendGitlab` posts:
  - a diff discussion reply: `POST projects/:id/merge_requests/:iid/discussions/:discussionId/notes`
    when the address names a discussion;
  - otherwise an Issue or MR note.
  Idempotency: body ends with the hidden marker `<!-- dsh-superagent:<message id> -->`; before
  posting, the newest 100 notes of the target are searched for the agent's note carrying that
  marker. `gone` for HTTP 404/403 (Issue/MR deleted or access revoked); other failures throw
  and are retried by the Outbox. `gitlabLink` returns the `web_url` (or `#iid`/`!iid` label).
- `src/sources/gitlab-read.ts`:
  - `threadContext`: reads notes of the Thread (system notes included as context) into
    `ContextMessage`; supports `thread` (paged), `root` (Issue/MR description) and `parent`
    (via `in_reply_to_id`); messages carry `path`/`line` for diff notes, `bot`, `url`.
  - `readGitlab(commands, resource)` for `orchestrator_read`: `gitlab/issues/<iid>`,
    `gitlab/issues/<iid>/comments/<page>`, `gitlab/merge_requests/<iid>`,
    `gitlab/merge_requests/<iid>/comments/<page>`, `gitlab/merge_requests/<iid>/thread/<note-id>`,
    `gitlab/merge_requests/<iid>/reviews/<page>` (approvals + review notes),
    `gitlab/merge_requests/<iid>/approvals`. 20 entries per page with `next_resource` /
    `next_offset`, matching the GitHub reader's shape.

### 7. MR binding and Ticket lifecycle (`src/sources/gitlab-mr.ts`)

Mirrors `github-pull.ts`:

- `agentMerge`: MR author is the deployment account and `source_project_id` is the
  deployment project.
- `ticketBranch`: the MR's `source_branch` last path segment equals the Ticket's name.
- `claimMerge`: an MR belongs to a Ticket when it is the Ticket's origin MR or an agent MR on
  the Ticket branch; an MR binds to at most one open Ticket.
- `observeMerge`: open MRs auto-bind to the single open Ticket on their branch; merged/closed
  MRs report `threadClosed('merged' | 'closed')` to the open Tickets following them; the
  Subscription remains for later human comments.
- Notices: a Ticket that ends because its MR merged/closed sends the existing chat notice for
  non-GitLab Channels, with GitLab wording (`MR !n 已合并`, `!n was closed without merging`).
  Generalize with a small dispatcher `changeEndNotices(ticket, request, deployment)` in
  `src/sources/end-notices.ts`; `github-pull.ts` keeps its function and `runtime.ts` calls the
  dispatcher instead.

### 8. Executor tools and runtime

- `runtime.ts`: expose a `gitlab` `SourceContext` beside `github`; `Prompts` receives the set of
  configured source kinds (not just `feishu`) so only configured Source skills are registered;
  the end-of-ticket notice uses the dispatcher.
- `tools/executor.ts`: keep the field name `pr_number` (prompts call it the Ticket's PR/MR) and
  dispatch by the Ticket's origin Trigger source: GitHub → `readPull`/`claimPull`; GitLab →
  `readMerge`/`claimMerge`. Tool descriptions and thread-key examples become source-neutral
  (`github:<number>` or `gitlab:mr:<iid>`).
- `app/source-context.ts`, `app/http.ts`: `accounts` gains `forge`
  (`'github'` | `'gitlab'`) and, for GitLab, `{ project, host, agent, commitEmail }`; existing
  GitHub fields stay for GitHub deployments. The panel's Deployment page shows both shapes.

### 9. Prompts and skills

- New built-in skill `skills/dsh-superagent-gitlab/SKILL.md` (packaged by
  `app/skill-files.ts`), documenting the `gitlab/…` resources, `glab` commands for comments and
  discussion replies, approval semantics, and that CI is not delivered yet.
- `app/context.ts`: `sourceSkills` gains `gitlab`; `app/prompts.ts` gains the part
  `skill/gitlab` and registers a built-in Source skill only when its Source is configured
  (`github`, `gitlab`, `feishu`); `composeSkill` handles the new name.
- Source-neutral wording in the built-in prompt text, so one text set serves both forges. The
  concrete replacements (files under `prompts/`):
  - `cli.md`: "Use the Source CLI (`gh` on GitHub, `glab` on GitLab) … the matching
    `dsh-superagent-gh/gitlab` skill names the commands"; keep the read-only rules.
  - `worker.md`, `orchestrator.md`: "Act only as the repository account the Execution
    environment names"; "use `gh` or `glab` per the repository's Source"; PR→"PR/MR"; branch
    rules unchanged.
  - `work.md`, `replies.md`, `dispatch.md`, `handoff*.md`,
    `team/communication.md`: "GitHub" → "the repository's forge" / "the source thread"; review
    thread wording covers GitLab discussions; disclosure line applies to both.
  - `team/delivery.md`: auto-merge example covers both CLIs
    (`glab mr merge <iid> --auto-merge` / the `gh` form); co-author trailer names the user-id
    lookup for each forge.
- `tests/cards.spec.ts` (asserted sentences) and `tests/model-surface.spec.ts` (size budgets)
  are updated with the new text.

### 10. Bundle, build, panel

- `cordis.patch.yml`: insert `dsh-superagent-gitlab` (`name: dsh-superagent/gitlab-watch`,
  `disabled: true`, like Feishu) so a GitHub installation is unchanged; the GitLab profile
  enables it and disables `dsh-superagent-github`.
- `tsdown.config.ts`: add the `gitlab-watch` entry; `package.json`: add the export and the
  `src/sources/gitlab.ts` file list is automatic.
- `examples/profile.cordis.patch.yml`: document the GitLab alternative next to Feishu.
- `tests/bundle-patch.spec.ts`: expected row ids gain the new entry.
- Panel: source settings form comes from the kind's `fields`; the Deployment page's account
  facts and setup problems need the new `gitlab` place. Locale entries (`locale/zh.json`,
  `locale/en.json`) gain the GitLab labels/messages.

### 11. Documentation

- `docs/001`: Source row and "what is reusable" table mention GitLab.
- `docs/002`: admission rows `issues`/`merges`/`comments` for GitLab.
- `docs/003`: execution environment variables for GitLab.
- `docs/004`: Source skills list.
- `docs/005`: panel's Source/Deployment descriptions.
- `docs/006`: prerequisites (`glab` 1.124+), Accounts (agent GitLab account + `glab-agent`
  wrapper), Settings (`source:gitlab`, policy, a GitLab prepare command), Setup checks
  (`glab` shim + git credential for the instance host), Try it, Troubleshooting.
- `docs/007`: module table (new files; updated responsibilities for commands, environment,
  deployment, resources, prompts, runtime) and test table; `docs/008` unchanged unless the
  new Source illustrates a better example.
- `README.md`: one line that GitHub and GitLab are supported.
- `SETUP.md`: the guided setup gains the GitLab path — Sources question offers GitLab (default
  for this operator), repository becomes `group/project`, agent account is a GitLab user,
  allowed list is GitLab usernames, checkout uses the GitLab URL, commit identity uses
  `_COMMIT_EMAIL`, auto-merge uses GitLab approvals/merge rules; operator steps cover creating
  the agent user, adding it to the project (Developer+), signing the wrapper in, and the
  project clone.

### 12. Installation on this machine (part of this work)

After the build passes:

1. Operator steps (only they can do these): create or choose the GitLab agent account and add
   it to the target project; create `~/.local/bin/glab-agent`
   (`GLAB_CONFIG_DIR="$HOME/.config/glab-agent"`) and run `glab-agent auth login --hostname
   gitlab.xiding.tech`; confirm `glab-agent api user` prints the agent.
2. Profile: `dsh superagent --from-default-profile web --dump-config`, then
   `dsh plugin --profile superagent add "link:<checkout>"`; machine config sets `stateDir`,
   enables `dsh-superagent-gitlab` and disables `dsh-superagent-github`, sets port 3082.
3. Start with `scripts/dsh-superagent start`, then save through the Management API:
   `source:gitlab` (`project`, `host`, `cli`, observe switches), `policy.sources.gitlab.allowed`,
   `executor` (prepare command cloning `https://<host>/<project>.git` and creating the
   `agent/$DSH_SUPERAGENT_NAME` worktree), and the prompts' Delivery requirements (commit
   author, merge rule). The install uses the operator's own account as the first allowed user;
   the agent's account is `_ACCOUNT`.
4. Verify: `scripts/dsh-superagent status` exits 0; `GET /sources` shows `gitlab` `ready`;
   comment `@<agent> <request>` on a test Issue → Ticket appears; the worker replies; it opens
   an MR from the ticket branch; an approval/review comment reaches it; merging ends the Ticket.

## Risks and mitigations

- **Polling cost / rate limits.** `updated_after` limits reads to changed items; notes reads are
  per changed Thread; the poll interval defaults to 60 s. A cap of 10 notes pages per Thread
  prevents unbounded catch-up; anything missed is re-read on the next round because cursors
  only advance after receipt.
- **GitLab CE API differences.** Approvals and `updated_after` were verified on the instance;
  CI, images and webhooks are out of scope. If an endpoint is missing on another instance, the
  Source reports `error` and the rest keeps working.
- **Prompt neutralization regressions.** `tests/cards.spec.ts`, `tests/model-surface.spec.ts`
  and a GitHub regression run guard the wording; behaviour-specific commands live in each
  skill, so a neutral sentence cannot lose a command the agent needs.
- **Duplicate replies after a restart.** Every send checks the marker; every Trigger id is
  derived from GitLab ids.
- **Wrong account acting.** Setup verifies the wrapper's account and push access; the worker
  environment shims `glab` and holds no operator token; `GITLAB_TOKEN`-shaped variables are
  cleared like `GH_TOKEN`.
- **Operator attention.** The install asks one question at a time and collects every operator
  step into one message, per `SETUP.md`'s philosophy.

## Verification and acceptance criteria

1. `pnpm check` (typecheck, tests, build) passes; the GitHub Source tests pass unchanged.
2. New `tests/gitlab.spec.ts` covers: trigger mapping for every event in scope; poll cursors and
   dedupe across rounds; scope/observation and admission rows; approval revision ids; channel
   idempotency and `gone`; thread context pages; MR claim/observe/merge notices; GitLab
   deployment resolution and its setup problems; environment variables.
3. `tests/docs.spec.ts`, `bundle-patch.spec.ts`, `cards.spec.ts`, `model-surface.spec.ts`
   updated and passing.
4. On this machine: setup resolves, `status` exits 0, `GET /sources` reports `gitlab` ready.
5. End-to-end on the operator's project: an Issue comment creates a Ticket, the agent
   acknowledges and answers in the Issue; an MR from the ticket branch binds; a review comment
   becomes an input to the worker; the merge ends the Ticket.

## Left to the builder

- Exact glab stderr shapes and the parsing fallback; pages of notes catch-up; polling interval
  and backoff constants; the precise layout of panel strings and locales.
- Which existing tests need text/budget updates, and the final wording of the neutral prompt
  sentences (subject to `cards.spec.ts`).
- The GitLab prepare command and Delivery requirements text for this installation (operator
  answers at setup time).

## Calls settled after the builder check

These were open in the draft; the builder can treat them as decided:

1. **Per-thread note cursor start.** A Thread's `KV.gitlab.cursor` starts at the stream's
   `since` (the installation or stream-start time) the first time the Thread is read, not at
   the current round. Triggers are deduplicated by id, so an overlap is harmless; starting at
   the stream cursor never loses notes written between the first stream read and the Thread's
   first note read.
2. **Bot detection.** A note author is a bot when `author.bot === true`; as a fallback for
   instances that omit the field, a username ending in `_bot` counts as a bot. Bots can only
   become inferred candidates and must still be listed under the Policy's `bots`.
3. **GitLab auto-merge.** The default Delivery requirements say a person merges: CI is out of
   scope, so "merge when pipeline succeeds" has no meaning yet. If an operator wants the agent
   to enable auto-merge anyway, the delivery text names `glab mr merge <iid> --auto-merge` and
   says a person's merge rule is unchanged. No auto-merge code is added.
4. **`GET state.accounts` stays backward compatible.** GitHub keeps the current shape and
   `github` fields; GitLab adds `forge: 'gitlab'` and a `gitlab` object. The panel reads by
   `forge`, so an installation never shows both.
5. **Prompt size budget.** Neutral wording must not grow a prompt part by more than about 5%;
   `tests/model-surface.spec.ts` budgets may be raised only for that headroom.
6. **glab error parsing.** `commandError` takes the HTTP status from `glab: <status> <reason>`
   or a JSON `{"message":"<status> ..."}` payload; anything else stays retryable, mirroring the
   GitHub transport.
7. **Approval triggers are subscription-only.** No admission row is added for approvals: an
   approval reaches a Ticket that already follows the MR, and never creates new work.

