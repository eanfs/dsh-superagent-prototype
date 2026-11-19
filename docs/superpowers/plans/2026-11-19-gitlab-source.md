# GitLab Source Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a `gitlab` Source to dsh-superagent (Issue create/close/comment, MR create/review/merge/comment), install the bundle on this machine, and run it against `gitlab.xiding.tech`.

**Architecture:** A new Source kind parallel to GitHub's: `gitlab-trigger` (schemas and Trigger builders), `gitlab-channel` (idempotent notes and discussion replies), `gitlab-read` (thread context and `orchestrator_read` resources), `gitlab-mr` (MR binding and Ticket end), and `gitlab` (kind definition plus polling loops). The deployment layer gains an optional `gitlab` account resolved through a `glab` wrapper, the worker environment gains a `glab` shim and GitLab git credentials, and the built-in prompts become forge-neutral with per-forge CLI detail moved into two skills. GitHub keeps working unchanged.

**Tech Stack:** TypeScript (ESM), zod, vitest, pnpm, tsdown, DeepSeek Harness plugin bundle, `glab` 1.124 and the GitLab REST API v4.

**Spec:** [docs/superpowers/specs/2026-11-19-gitlab-source-design.md](../specs/2026-11-19-gitlab-source-design.md)

## Global Constraints

- GitLab instance: `https://gitlab.xiding.tech` (API v4), CE 19.5.1; `glab` 1.124.0 signed in per host. No other instance is required to work.
- One work repository per installation. Both a GitHub repository and a GitLab project configured is a setup problem (`repositoryAmbiguous`), not a merge.
- Every GitLab API path is project-scoped as `projects/<encodeURIComponent(path_with_namespace)>/...`; absolute paths, schemes and `..` are refused, exactly like the GitHub transport.
- Push access means `permissions.project_access.access_level >= 30` or `permissions.group_access.access_level >= 30`.
- Thread keys: `gitlab:issue:<iid>` and `gitlab:mr:<iid>` (Issue and MR iids are separate number spaces).
- Comment idempotency marker: `<!-- dsh-superagent:<message id> -->` at the end of posted bodies.
- Note catch-up cap: 10 pages of 100 notes per Thread per round; exceeding it is a round failure, never a silent drop.
- Cursor overlap: new cursors are set to the newest observed timestamp minus 5 s (`cursorAt`); stable Trigger ids absorb the overlap.
- GitLab Source config defaults: `pollSeconds` 60 (allowed 60–3600), `failuresBeforeError` 3 (1–20), `observeIssues`/`observeMerges`/`observeComments` false, `host` `gitlab.com`.
- Execution environment variables: `DSH_SUPERAGENT_GITLAB_PROJECT`, `_HOST`, `_ACCOUNT`, `_COMMIT_EMAIL`, `_KIND` (`issue`/`mr`), `_IID`, `_SOURCE_BRANCH`.
- Trigger ids: `key('gitlab','issue',iid)`, `key('gitlab','mr',iid)`, `key('gitlab','note',note.id)`, `key('gitlab','issue-state',iid,state,updated_at)`, `key('gitlab','mr-state',iid,state,updated_at)`, `key('gitlab','approval',iid,revision)`.
- Built-in prompt text names no forge-only command: `gh` and pull-request specifics stay in the GitHub skill, `glab` and MR specifics in the new GitLab skill.
- Every new file under `src/` and `tests/` is listed in the module and test tables of `docs/007-architecture.md` in the same task that creates it (`tests/docs.spec.ts` enforces this). Every new class gets a JSDoc block (`tests/class-docs.spec.ts`).
- Each task leaves `pnpm check` (typecheck, vitest, build) green and ends in one commit with a conventional message.

## Review Focus

1. **A round aborted or the process restarted mid-poll must not duplicate replies or lose events.** Expected: cursors advance only after `receive`, a thrown round leaves them untouched, and re-reading the overlap yields `duplicate`. Pinned by T9's "keeps cursors when a round is aborted" test.
2. **The agent's own comments and GitLab system notes must never start or wake work.** Expected: notes authored by the deployment account, and `system: true` notes, produce no Trigger. Pinned by T4's translation test and T9's poll test.
3. **More than 1,000 new notes on one Issue/MR between polls must surface as an error, not vanish.** Expected: the round throws, the per-thread cursor stays, health reports `pollFailures` after the threshold. Pinned by T9's "fails instead of dropping notes past the catch-up cap" test.
4. **An uncertain POST (GitLab accepted it, the CLI failed afterwards) must not duplicate the comment.** Expected: the marker is found in the target's recent notes and the send reports `sent`. Pinned by T6's "reconciles an uncertain GitLab send" test.
5. **Case and shape differences in identities must not split one identity.** Expected: agent username, allowed-list usernames, project paths and usernames compare case-insensitively through `sameName`. Pinned by T1's canonical-project test and T4's `agent`/mention test.

---

### Task 1: Deployment supports a GitLab account

**Files:**
- Modify: `src/deployment.ts` (account schemas, `Deployment`, `SETUP_FIELDS`, `resolveGitlabDeployment`, `resolveForge`)
- Modify: `src/setup.ts` (`SetupPlace`, new `SETUP_PROBLEMS` entries)
- Modify: `src/app/setup.ts` (`resolve` uses `resolveForge`, `checkAccounts` dispatches `source:gitlab`)
- Modify: `src/app/http.ts` (`setupView().accounts` reads the resolved forge)
- Modify: `tests/fixtures/deployment.ts` (`Deployment` now has optional `github`; add
  `export const testGitlabAccount = (overrides: Partial<GitlabAccount> = {}): GitlabAccount => ({ cli: '/bin/false', host: 'gitlab.example.com', login: 'agent-bot', userId: 7, commitEmail: '7-agent-bot@example.com', project: 'group/widgets', ...overrides })` and
  `export const testGitlabDeployment = (overrides: Partial<Deployment> = {}): Deployment => ({ ...testMachine(), repository: 'group/widgets', gitlab: testGitlabAccount(), ...overrides })`)
- Test: `tests/deployment.spec.ts`

**Interfaces:**
- Consumes: `Machine`, `Probe`, `executable`, `SetupError`.
- Produces:
  ```ts
  export const GitlabAccountSchema: z.ZodType<{ project: string; host: string; cli: string }>
  export interface GitlabAccount { cli: string; host: string; login: string; userId: number; commitEmail: string; project: string }
  export interface Deployment extends Machine { repository: string; github?: { cli: string; login: string; noreply: string }; gitlab?: GitlabAccount; feishu?: FeishuBot }
  export const SETUP_FIELDS: Readonly<Record<'github' | 'gitlab' | 'feishu', readonly string[]>>
  export async function resolveGitlabDeployment(machine: Machine, gitlab: unknown, probe: Probe, path?: string): Promise<Deployment>
  export async function resolveForge(machine: Machine, sources: { github?: unknown; gitlab?: unknown }, probe: Probe, path?: string): Promise<Deployment>
  ```
  Setup problems: place `'gitlab'`; codes `projectUnset`, `gitlabCliUnset`, `gitlabCliMissing`, `gitlabCliSignedOut`, `gitlabCliNoAccount`, `projectUnreadable`, `projectUnreported`, `projectReadOnly`, `repositoryAmbiguous` (place `'deployment'`, text "This installation names both a GitHub repository and a GitLab project; keep one work repository.").

- [ ] **Step 1: Write the failing tests**

Append to `tests/deployment.spec.ts`:

```ts
/** Fake glab CLI signed in as `login` with the given project access level. */
const glab = (file: string, login = 'Agent-Bot', level = 30, path = 'group/widgets'): string => program(file,
  `case "$*" in
     "api --hostname gitlab.example.com user") echo '{"id":7,"username":"${login}","commit_email":"${login}@example.com"}' ;;
     "api --hostname gitlab.example.com projects/group%2Fwidgets") echo '{"path_with_namespace":"${path}","permissions":{"project_access":{"access_level":${level}}}}' ;;
     *) exit 1 ;;
   esac`)

it('reads the GitLab project and agent account from glab, canonicalised', async () => {
  const deployment = await resolveGitlabDeployment(machine(), { project: 'GROUP/Widgets', host: 'gitlab.example.com', cli: 'glab' }, probe, path)
  expect(deployment).toMatchObject({ repository: 'group/widgets', gitlab: { login: 'Agent-Bot', userId: 7,
    commitEmail: 'Agent-Bot@example.com', host: 'gitlab.example.com', project: 'group/widgets' } })
  expect(deployment.github).toBeUndefined()
})

it('names what GitLab setup needs and refuses a read-only project', async () => {
  const needs = (place: string, code: string, values: Record<string, unknown>, gitlab: unknown) =>
    expect(resolveGitlabDeployment(machine(), gitlab, probe, path)).rejects.toEqual(setupError(place, code, values))
  const project = 'group/widgets', host = 'gitlab.example.com'
  await needs('gitlab', 'projectUnset', {}, { cli: 'glab', host })
  await needs('gitlab', 'gitlabCliUnset', {}, { project, host })
  await needs('gitlab', 'projectReadOnly', { account: 'Agent-Bot', repository: 'group/widgets' },
    { project, host, cli: glab(join(dir, 'bin', 'glab-reader'), 'Agent-Bot', 20) })
})

it('refuses one installation that configures two work repositories', async () => {
  await expect(resolveForge(machine(), { github: { repository: 'acme/widgets', cli: 'gh' }, gitlab: { project: 'group/widgets', cli: 'glab' } }, probe, path))
    .rejects.toEqual(setupError('deployment', 'repositoryAmbiguous'))
})
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `pnpm vitest run tests/deployment.spec.ts -t 'GitLab'`
Expected: FAIL, `resolveGitlabDeployment is not a function` / unknown setup code.

- [ ] **Step 3: Implement in `src/deployment.ts` and `src/setup.ts`**

- `GitlabAccountSchema` (strict): `project` `group/project` (letters, digits, `_.-` per segment, subgroups allowed), `host` default `gitlab.com`, `cli` default `''`.
- `resolveGitlabDeployment`: require project and CLI; resolve the executable with `executable(cli, path, workerBin(stateDir))`; probe `[cli, 'api', '--hostname', host, 'user']` and `[cli, 'api', '--hostname', host, `projects/${encodeURIComponent(project)}`]`; parse with zod; canonicalise `project` to `path_with_namespace`, `repository` to the same; login from `username`, `commitEmail` from `commit_email`; check access level.
- `resolveForge`: parse both configs, throw `repositoryAmbiguous` when both name a repository, otherwise delegate to `resolveGitlabDeployment` or `resolveDeployment`.
- `src/app/setup.ts`: call `resolveForge(machine, { github: settings.sources.github?.config, gitlab: settings.sources.gitlab?.config }, probe)`; `checkAccounts` maps `source:gitlab` to `resolveGitlabDeployment`.
- `src/app/http.ts` `setupView`: `accounts = { repository, agent: resolved.gitlab?.login ?? resolved.github?.login ?? '', forge: resolved.gitlab ? 'gitlab' : 'github', feishu: … }`.
- Update every `deployment.github.<x>` compile site flagged by `pnpm typecheck` with an explicit guard (T2/T3/T7 will replace the guards with real behaviour).

- [ ] **Step 4: Run tests and typecheck**

Run: `pnpm vitest run tests/deployment.spec.ts` then `pnpm typecheck`
Expected: PASS (the guards added in Step 3 keep typecheck green).

- [ ] **Step 5: Commit**

```bash
git add src/deployment.ts src/setup.ts src/app/setup.ts src/app/http.ts tests/deployment.spec.ts tests/fixtures/deployment.ts
git commit -m "feat: resolve a GitLab project and agent account during setup"
```

---

### Task 2: GitLab transport in `Commands`

**Files:**
- Modify: `src/infra/commands.ts`
- Test: `tests/commands.spec.ts`

**Interfaces:**
- Consumes: `deployment.gitlab` from Task 1.
- Produces:
  ```ts
  async glab<T>(args: string[]): Promise<T>
  async gitlab<T>(schema: z.ZodType<T>, path: string, ...flags: string[]): Promise<T>
  async gitlabApi(path: string, ...flags: string[]): Promise<string>
  async gitlabPages<T>(item: z.ZodType<T>, path: string, tally?: Tally): Promise<T[]>
  ```
  Every method scope-checks the path and throws `Error('GitLab calls are restricted to the configured project')` for `/...`, `scheme:...`, `..`, like `Commands.path`. `gitlab`/`gitlabApi`/`gitlabPages` build `['api','--hostname',deployment.gitlab.host, path, ...flags]`; `gitlabPages` adds `--paginate --output json` and validates with `validItems`.

- [ ] **Step 1: Write the failing tests**

In `tests/commands.spec.ts` add `const gitlabDeployment = testGitlabDeployment({ gitlab: testGitlabAccount({ cli: '/path/agent-glab' }) })` and:

```ts
it('scopes GitLab calls to the configured project and host', async () => {
  const commands = new Commands(() => gitlabDeployment, {}, new AbortController().signal); const run = vi.spyOn(commands, 'run').mockResolvedValue('{}')
  await commands.gitlab(z.unknown(), 'issues/1')
  expect(run).toHaveBeenCalledWith('/path/agent-glab', ['api', '--hostname', 'gitlab.example.com', 'projects/group%2Fwidgets/issues/1'])
  for (const path of ['../other/repo/issues', 'issues/../../x', '/projects/1', 'https://gitlab.example.com/x'])
    await expect(commands.gitlabApi(path)).rejects.toThrow('GitLab calls are restricted to the configured project')
})
it('reads GitLab pages as one array and skips malformed items', async () => {
  const commands = new Commands(() => testGitlabDeployment(), {}, new AbortController().signal)
  vi.spyOn(commands, 'run').mockResolvedValue('[{"id":1},{"bad":true}]')
  expect(await commands.gitlabPages(z.looseObject({ id: z.number() }), 'issues')).toEqual([{ id: 1 }])
  expect(vi.spyOn(commands, 'run').mock.calls[0]![1]).toContain('--paginate')
})
it('carries a GitLab HTTP status in CliError', () => {
  expect(commandError({ stderr: 'glab: 404 Not Found', code: 1 }).code).toBe(404)
  expect(commandError({ stderr: '{"message":"404 Project Not Found"}', code: 1 }).code).toBe(404)
})
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `pnpm vitest run tests/commands.spec.ts`
Expected: FAIL, `commands.gitlab is not a function`.

- [ ] **Step 3: Implement the four methods and extend `commandError`**

`commandError` gains a GitLab branch after the `gh` branch: parse JSON `{"message":"<status> ..."}` or a `glab:` line; take the first 4xx/5xx status it contains; return `new CliError(message, status, exitCode)`; fall through to the existing behaviour otherwise. `gh`/`api`/`pages` throw `'The GitLab Source has no working glab CLI'`-style errors only when called without a GitLab deployment.

- [ ] **Step 4: Run tests and typecheck**

Run: `pnpm vitest run tests/commands.spec.ts` and `pnpm typecheck`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/infra/commands.ts tests/commands.spec.ts
git commit -m "feat: add a GitLab CLI transport scoped to one project"
```

---

### Task 3: Worker environment for GitLab

**Files:**
- Modify: `src/infra/environment.ts` (`WORKER_COMMANDS`, `agentVariables`, `writeEnvironment`, `verifyEnvironment`, blocked-command text)
- Test: `tests/environment.spec.ts`

**Interfaces:**
- Consumes: `Deployment` with optional `github`/`gitlab` from Task 1.
- Produces:
  ```ts
  export const WORKER_COMMANDS: readonly string[] // ['gh', 'glab', 'git', 'lark-cli']
  export function agentVariables(stateDir: string): Record<string, string | undefined> // adds GITLAB_TOKEN: undefined, GITLAB_ACCESS_TOKEN: undefined
  export async function writeEnvironment(deployment: Pick<Deployment, 'stateDir' | 'repository' | 'github' | 'gitlab' | 'feishu'>, blockedCommands: readonly string[]): Promise<void>
  export async function verifyEnvironment(deployment: Pick<Deployment, 'github' | 'gitlab' | 'stateDir'>, subprocess: SubprocessRuntime, signal: AbortSignal): Promise<void>
  ```
  `worker-gitconfig` for a GitLab deployment contains, instead of the github.com entries:
  ```ini
  [credential]
  	helper =
  [credential "https://gitlab.example.com"]
  	helper = "!/path/agent-glab auth git-credential"
  [url "https://gitlab.example.com/"]
  	insteadOf = git@gitlab.example.com:
  	insteadOf = ssh://git@gitlab.example.com/
  ```
  plus the existing `[user] useConfigOnly` and LFS filters. `worker-bin/glab` is written only for a GitLab deployment (and `gh` only for GitHub); other shims are unchanged.

- [ ] **Step 1: Write the failing tests**

In `tests/environment.spec.ts` add a GitLab deployment (`testDeployment({ github: undefined, gitlab: { … } })`) and:

```ts
it('writes a glab shim and the GitLab credential helper, and no gh shim', async () => {
  const deployment = testGitlabDeployment()
  await writeEnvironment(deployment, [])
  expect(readFileSync(join(workerBin(deployment.stateDir), 'glab'), 'utf8')).toContain('exec')
  expect(existsSync(join(workerBin(deployment.stateDir), 'gh'))).toBe(false)
  const config = readFileSync(workerGitConfig(deployment.stateDir), 'utf8')
  expect(config).toContain('[credential "https://gitlab.example.com"]')
  expect(config).toContain('auth git-credential')
  expect(config).not.toContain('github.com')
})
it('removes GitLab token variables from the agent environment', () => {
  expect(agentVariables('/tmp/state')).toMatchObject({ GITLAB_TOKEN: undefined, GITLAB_ACCESS_TOKEN: undefined })
})
it('blocks a command by naming the forge CLI', async () => {
  await writeEnvironment(testGitlabDeployment(), ['glab-personal'])
  expect(readFileSync(join(workerBin(testGitlabDeployment().stateDir), 'glab-personal'), 'utf8')).toContain('glab')
})
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `pnpm vitest run tests/environment.spec.ts`
Expected: FAIL, no `glab` shim.

- [ ] **Step 3: Implement the environment changes**

Keep the existing marker/foreign-file/atomic-write machinery. Build the shim list and gitconfig entries from the configured forge; blocked-command text becomes ```${name} is blocked in this installation; use the repository CLI, which acts as the agent account.``` `verifyEnvironment` checks the helper of each configured forge (GitHub: existing key; GitLab: `credential.https://<host>.helper`) and keeps the `gitTooOld` setup error, whose text Task 12 rewords.

- [ ] **Step 4: Run tests and typecheck**

Run: `pnpm vitest run tests/environment.spec.ts` and `pnpm typecheck`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/infra/environment.ts tests/environment.spec.ts
git commit -m "feat: give workers a glab shim and GitLab credentials"
```

---

### Task 4: GitLab schemas and Triggers

**Files:**
- Create: `src/sources/gitlab-trigger.ts`
- Create: `tests/gitlab.spec.ts`
- Modify: `docs/007-architecture.md` (Modules row for the new module; Tests row for `tests/gitlab.spec.ts`)

**Interfaces:**
- Consumes: `key`, `Trigger`, `ChannelRef`, `sameName`.
- Produces:
  ```ts
  export const SOURCE = 'gitlab'
  export const UserSchema, IssueSchema, MergeRequestSchema, NoteSchema // zod looseObject, fields listed below
  export type GitlabUser, GitlabIssue, GitlabMergeRequest, GitlabNote
  export const isBot: (author: GitlabUser) => boolean // author.bot === true || /_bot$/i.test(author.username)
  export const mentions: (text: string | null | undefined, username: string) => boolean
  export const issueThread: (iid: number) => ThreadRef   // { source: SOURCE, id: 'issue:<iid>' }
  export const mergeThread: (iid: number) => ThreadRef   // { source: SOURCE, id: 'mr:<iid>' }
  export const threadOf: (kind: 'issue' | 'mr', iid: number) => ThreadRef
  export const issueTrigger: (issue: GitlabIssue, agent: string) => Trigger
  export const mergeTrigger: (mr: GitlabMergeRequest, agent: string) => Trigger
  export const issueStateTrigger: (issue: GitlabIssue, state: 'closed' | 'reopened') => Trigger
  export const mergeStateTrigger: (mr: GitlabMergeRequest, state: 'closed' | 'reopened') => Trigger
  export const noteTrigger: (note: GitlabNote, ref: { kind: 'issue' | 'mr'; iid: number }, agent: string, urlBase: string) => Trigger | undefined
  export const approvalTrigger: (mr: GitlabMergeRequest, revision: number, approvedBy: string[], approved: boolean, url: string) => Trigger
  export const gitlabChannel: (kind: 'issue' | 'mr', iid: number, extra?: { discussionId?: string; noteId?: number; url?: string }) => ChannelRef
  export const findThreads: (query: string) => { threads: string[]; terms: string[] }
  ```
  Schema fields: `IssueSchema` `{ id, iid, title, description: string|null, state: string, created_at, updated_at, closed_at?: string|null, author: UserSchema, web_url }`; `MergeRequestSchema` the same plus `source_branch, target_branch, source_project_id, merged_at?, draft?, work_in_progress?, sha?`; `NoteSchema` `{ id, body, author: UserSchema, created_at, updated_at?: string|null, system: boolean (default false), type?: string|null, discussion_id?: string|null, in_reply_to_id?: number|null, resolvable?: boolean, resolved?: boolean, position?: { new_path?: string, new_line?: number, old_line?: number } | null }`. Trigger `meta`: `{ kind, iid, url, noteId?, discussionId?, replyTo?, path?, line? }`; `text` is `description`/`body`; `addressed: 'mention'` when `mentions` is true; `human: !isBot(author)`; channel comes from `gitlabChannel`; `repliesTo` uses `in_reply_to_id` → `key('gitlab','note',in_reply_to_id)`.

- [ ] **Step 1: Write the failing tests**

Create `tests/gitlab.spec.ts` with translation tests:

```ts
it('translates an Issue, its close and its comments, dropping the agent and system notes', () => {
  const issue = { id: 1, iid: 12, title: 'Fix', description: 'Please @agent-bot fix it', state: 'opened', created_at: '2026-11-19T01:00:00Z',
    updated_at: '2026-11-19T01:00:00Z', author: { id: 2, username: 'alice' }, web_url: 'https://gitlab.example.com/group/widgets/-/issues/12' }
  expect(issueTrigger(issue, 'Agent-Bot')).toMatchObject({ id: key('gitlab', 'issue', 12), source: 'gitlab', addressed: 'mention', human: true,
    threads: [{ source: 'gitlab', id: 'issue:12' }], actor: { source: 'gitlab', id: 'alice' } })
  expect(issueStateTrigger({ ...issue, state: 'closed' }, 'closed').id).toBe(key('gitlab', 'issue-state', 12, 'closed', issue.updated_at))
  const note = (over: object) => ({ id: 5, body: 'hi @agent-bot', author: { id: 2, username: 'alice' }, created_at: '2026-11-19T01:05:00Z',
    system: false, discussion_id: 'd1', noteable_iid: 12, ...over })
  expect(noteTrigger(note({}), { kind: 'issue', iid: 12 }, 'agent-bot', issue.web_url)!).toMatchObject({ id: key('gitlab', 'note', 5), addressed: 'mention' })
  expect(noteTrigger(note({ author: { id: 7, username: 'agent-bot' } }), { kind: 'issue', iid: 12 }, 'agent-bot', issue.web_url)).toBeUndefined()
  expect(noteTrigger(note({ system: true }), { kind: 'issue', iid: 12 }, 'agent-bot', issue.web_url)).toBeUndefined()
  expect(noteTrigger(note({ body: 'plain' }), { kind: 'issue', iid: 12 }, 'agent-bot', issue.web_url)!.addressed).toBeUndefined()
})
it('translates an MR, a diff comment, an approval and a merge state', () => {
  const mr = { id: 9, iid: 3, title: 'Fix', description: null, state: 'opened', created_at: '2026-11-19T02:00:00Z', updated_at: '2026-11-19T02:00:00Z',
    source_branch: 'agent/fix', target_branch: 'main', source_project_id: 413, author: { id: 2, username: 'alice' },
    web_url: 'https://gitlab.example.com/group/widgets/-/merge_requests/3' }
  expect(mergeTrigger(mr, 'agent-bot')).toMatchObject({ id: key('gitlab', 'mr', 3), threads: [{ source: 'gitlab', id: 'mr:3' }] })
  const diff = { id: 6, body: 'reply', author: { id: 2, username: 'alice', bot: false }, created_at: '2026-11-19T02:10:00Z', system: false,
    type: 'DiffNote', discussion_id: 'dd', in_reply_to_id: 4, position: { new_path: 'src/a.ts', new_line: 4 } }
  expect(noteTrigger(diff, { kind: 'mr', iid: 3 }, 'agent-bot', mr.web_url)!.meta).toMatchObject({ kind: 'mr', iid: 3, discussionId: 'dd', path: 'src/a.ts', line: 4 })
  expect(approvalTrigger(mr, 2, ['alice'], true, mr.web_url)).toMatchObject({ id: key('gitlab', 'approval', 3, 2), human: true })
})
it('finds Issue and MR threads from query conventions', () => {
  expect(findThreads('#12').threads).toEqual(['gitlab:issue:12'])
  expect(findThreads('!3').threads).toEqual(['gitlab:mr:3'])
})
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `pnpm vitest run tests/gitlab.spec.ts`
Expected: FAIL, module not found.

- [ ] **Step 3: Implement `src/sources/gitlab-trigger.ts`**

Use `z.looseObject`; validate nothing outside the builders; derive ids with `key`; compare usernames with `sameName`; `mentions` matches `@username` at a username boundary case-insensitively.

- [ ] **Step 4: Run tests and typecheck**

Run: `pnpm vitest run tests/gitlab.spec.ts` and `pnpm typecheck`
Expected: PASS.

- [ ] **Step 5: Update the 007 tables and commit**

Add the Modules row for `src/sources/gitlab-trigger.ts` and the Tests row for `tests/gitlab.spec.ts`; run `pnpm vitest run tests/docs.spec.ts` (PASS).

```bash
git add src/sources/gitlab-trigger.ts tests/gitlab.spec.ts docs/007-architecture.md
git commit -m "feat: translate GitLab events into Triggers"
```

---

### Task 5: The `gitlab` kind: config, scope, policy, environment

**Files:**
- Create: `src/sources/gitlab.ts`
- Modify: `src/kv.ts` (only the `KV.gitlab` families needed by later tasks)
- Test: `tests/gitlab.spec.ts` (append)
- Modify: `docs/007-architecture.md` (Modules row)

**Interfaces:**
- Consumes: `SourceKind`, `key`, Task 4's builders.
- Produces:
  ```ts
  export const GitlabConfigSchema: z.ZodType<GitlabConfig>
  export interface GitlabConfig { project: string; host: string; cli: string; pollSeconds: number; failuresBeforeError: number;
    observeIssues: boolean; observeMerges: boolean; observeComments: boolean }
  export const gitlabKind: SourceKind<GitlabConfig>
  export const name = 'dsh-superagent-gitlab'
  export const inject = ['orchestrator']
  export function apply(ctx: Context): void // registers the kind; Task 9 adds the polling loop
  ```
  `gitlabKind` fields: `project`, `host`, `cli` (text), `pollSeconds`, `failuresBeforeError` (number, advanced), `observeIssues`, `observeMerges`, `observeComments` (switch). `health: ['gitlab']`. `scope` maps `meta.kind === 'issue' → 'issues'`, `'mr' → 'merges'`, a note/comment → `'comments'`, returning `{ listened: true, observed, row }`. Policy `allowed` ids `^[A-Za-z0-9_.-]+$`, optional `bots`; rows `issues`, `merges`, `comments` with cells driven by the observe switches exactly like GitHub's `rows`. `environment.variables/values` use the Global Constraints names. `facts` shows `当前账号 @<login>`.

- [ ] **Step 1: Write the failing tests**

Append to `tests/gitlab.spec.ts`:

```ts
it('applies defaults, scope rows and observation switches', () => {
  const config = GitlabConfigSchema.parse({ project: 'group/widgets' })
  expect(config).toEqual({ project: 'group/widgets', host: 'gitlab.com', cli: '', pollSeconds: 60, failuresBeforeError: 3,
    observeIssues: false, observeMerges: false, observeComments: false })
  expect(gitlabKind.scope!({ meta: { kind: 'issue' } }, config)).toEqual({ listened: true, observed: false, row: 'issues' })
  expect(gitlabKind.scope!({ meta: { kind: 'mr' } }, { ...config, observeMerges: true })).toEqual({ listened: true, observed: true, row: 'merges' })
  expect(gitlabKind.policy!.rows!(config).map(row => row.id)).toEqual(['issues', 'merges', 'comments'])
})
it('supplies the GitLab execution variables for Issue and MR tickets', () => {
  const deployment = testDeployment({ github: undefined, gitlab: { cli: '/bin/glab', host: 'gitlab.example.com', login: 'agent-bot',
    userId: 7, commitEmail: '7-agent-bot@example.com', project: 'group/widgets' } })
  const config = GitlabConfigSchema.parse({ project: 'group/widgets', host: 'gitlab.example.com' })
  expect(gitlabKind.environment!.values({ deployment, config })).toMatchObject({ DSH_SUPERAGENT_GITLAB_PROJECT: 'group/widgets',
    DSH_SUPERAGENT_GITLAB_ACCOUNT: 'agent-bot', DSH_SUPERAGENT_GITLAB_IID: '' })
  const trigger = mergeTrigger({ id: 9, iid: 3, title: '', description: null, state: 'opened', created_at: '', updated_at: '',
    source_branch: 'agent/fix', target_branch: 'main', source_project_id: 413, author: { id: 2, username: 'alice' },
    web_url: 'https://gitlab.example.com/group/widgets/-/merge_requests/3' }, 'agent-bot')
  expect(gitlabKind.environment!.values({ deployment, config, trigger })).toMatchObject({ DSH_SUPERAGENT_GITLAB_KIND: 'mr',
    DSH_SUPERAGENT_GITLAB_IID: '3', DSH_SUPERAGENT_GITLAB_SOURCE_BRANCH: 'agent/fix' })
})
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `pnpm vitest run tests/gitlab.spec.ts -t 'GitLab execution variables'`
Expected: FAIL, module not found.

- [ ] **Step 3: Implement the kind**

- Add `KV.gitlab` with `stream(kind)`, `cursor(thread)`, `state(thread)`, `approvals(iid)` and their value types.
- Config description strings name what each switch does, in both languages, matching GitHub's copy with GitLab words.
- `apply` calls `ctx.effect(() => ctx.orchestrator.sources.register(gitlabKind), 'gitlab: source kind')` only; Task 9 adds the loop and the health entry's periodic report.
- `environment.values`: empty values when the trigger is from another Source.

- [ ] **Step 4: Run tests and typecheck**

Run: `pnpm vitest run tests/gitlab.spec.ts` and `pnpm typecheck`
Expected: PASS.

- [ ] **Step 5: Update the 007 Modules row and commit**

```bash
git add src/sources/gitlab.ts src/kv.ts tests/gitlab.spec.ts docs/007-architecture.md
git commit -m "feat: define the GitLab Source kind"
```

---

### Task 6: GitLab Channel

**Files:**
- Create: `src/sources/gitlab-channel.ts`
- Modify: `src/sources/gitlab.ts` (add `channel`)
- Test: `tests/gitlab.spec.ts` (append)
- Modify: `docs/007-architecture.md` (Modules row)

**Interfaces:**
- Consumes: `Commands.gitlabApi/gitlabPages`, `OutboundMessage`, `ChannelLink`, Task 4's `GitlabNote`.
- Produces:
  ```ts
  export async function sendGitlab(host: Pick<SourceContext, 'commands'>, message: OutboundMessage): Promise<'sent' | 'gone'>
  export function gitlabLink(address: Record<string, string>, links: ChannelLinks): ChannelLink
  ```
  Address fields: `kind` (`issue`/`mr`), `iid`, optional `discussionId`, optional `url`. POST target: `issues/<iid>/notes` or `merge_requests/<iid>/notes`, or `merge_requests/<iid>/discussions/<discussionId>/notes`; body is `message.text + '\n\n<!-- dsh-superagent:' + message.id + ' -->'`. Before POST, read the newest 100 notes of the target and return `sent` when the agent's note already carries the marker. `gone` for `CliError` 403/404; anything else throws.

- [ ] **Step 1: Write the failing tests**

```ts
const gitlabReply = (over: Partial<OutboundMessage> = {}): OutboundMessage => ({ id: 'stable', kind: 'reply', text: 'Done',
  channel: gitlabChannel('mr', 3, { discussionId: 'dd' }), ...over })
it('reconciles an uncertain GitLab send and posts once', async () => {
  const commands = new Commands(() => testGitlabDeployment(), {}, new AbortController().signal)
  const pages = vi.spyOn(commands, 'gitlabPages').mockResolvedValueOnce([{ id: 1, body: 'Done\n\n<!-- dsh-superagent:stable -->', author: { id: 7, username: 'agent-bot' }, created_at: '', system: false }])
  const api = vi.spyOn(commands, 'gitlabApi').mockResolvedValue('{}')
  expect(await sendGitlab({ commands }, gitlabReply())).toBe('sent'); expect(api).not.toHaveBeenCalled()
  pages.mockResolvedValueOnce([])
  await sendGitlab({ commands }, gitlabReply())
  expect(api).toHaveBeenCalledWith('merge_requests/3/discussions/dd/notes', '--method', 'POST', '-f', expect.stringMatching(/Done\n\n<!-- dsh-superagent:stable -->$/))
})
it('reports gone only for a missing or forbidden target', async () => {
  const commands = new Commands(() => testGitlabDeployment(), {}, new AbortController().signal)
  vi.spyOn(commands, 'gitlabPages').mockResolvedValue([])
  vi.spyOn(commands, 'gitlabApi').mockRejectedValue(new CliError('404 Not Found', 404))
  expect(await sendGitlab({ commands }, gitlabReply())).toBe('gone')
  vi.spyOn(commands, 'gitlabApi').mockRejectedValue(new Error('network'))
  await expect(sendGitlab({ commands }, gitlabReply())).rejects.toThrow('network')
})
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `pnpm vitest run tests/gitlab.spec.ts -t 'uncertain GitLab send'`
Expected: FAIL, module not found.

- [ ] **Step 3: Implement `gitlab-channel.ts` and wire `gitlabKind.channel`**

Use `z.looseObject({ id: z.number(), body: z.string().nullish(), author: z.looseObject({ username: z.string() }) })` for the idempotency read; compare with `sameName`.

- [ ] **Step 4: Run tests and typecheck**

Run: `pnpm vitest run tests/gitlab.spec.ts` and `pnpm typecheck`
Expected: PASS.

- [ ] **Step 5: Update 007 and commit**

```bash
git add src/sources/gitlab-channel.ts src/sources/gitlab.ts tests/gitlab.spec.ts docs/007-architecture.md
git commit -m "feat: reply on GitLab Issues and MR discussions"
```

---

### Task 7: Thread context and `orchestrator_read` resources

**Files:**
- Create: `src/sources/gitlab-read.ts`
- Modify: `src/sources/gitlab.ts` (add `threadContext`, `find`)
- Modify: `src/app/resources.ts` (`RESOURCES` entry, `gitlab/` route, source label in the offset error)
- Create: `tests/gitlab-read.spec.ts`
- Modify: `docs/007-architecture.md` (Modules and Tests rows)

**Interfaces:**
- Consumes: `Commands`, `Trigger`, `ThreadContext`, `ContextPart`.
- Produces:
  ```ts
  export function gitlabRef(trigger: Trigger): { kind: 'issue' | 'mr'; iid: number } | undefined // host and project come from the kind config or from commands.deployment.gitlab
  export const gitlabThreadContext: ThreadContext<GitlabConfig>
  export async function readGitlab(commands: Commands, resource: string): Promise<unknown>
  ```
  Thread context parts: `thread` reads notes `?per_page=20&page=<cursor ?? 1>&sort=asc&order_by=created_at`; `root` returns the Issue/MR description as one `ContextMessage`; `parent` reads `notes/<in_reply_to_id>`. `ContextMessage` fields `{ id, from, at, text, bot?, reply_to?, url?, path?, line? }`. Resource names: `gitlab/issues/<iid>`, `gitlab/issues/<iid>/comments/<page>`, `gitlab/merge_requests/<iid>`, `gitlab/merge_requests/<iid>/comments/<page>`, `gitlab/merge_requests/<iid>/thread/<note-id>`, `gitlab/merge_requests/<iid>/reviews/<page>`, `gitlab/merge_requests/<iid>/approvals`; 20 entries per page with `next_offset`.

- [ ] **Step 1: Write the failing tests**

```ts
it('reads a GitLab Thread page and its root as context messages', async () => {
  const commands = new Commands(() => testGitlabDeployment(), {}, new AbortController().signal)
  vi.spyOn(commands, 'gitlabPages').mockResolvedValueOnce([{ id: 1, body: 'first', author: { id: 2, username: 'alice' }, created_at: '2026-11-19T01:00:00Z', system: false }])
  const page = await gitlabThreadContext.read(commands, mrTriggerFixture, 'thread', undefined, GitlabConfigSchema.parse({ project: 'group/widgets' }))
  expect(page.messages).toEqual([{ id: '1', from: 'alice', at: '2026-11-19T01:00:00Z', text: 'first' }])
  expect(page.next).toBe('2')
})
it('serves GitLab review resources for orchestrator_read', async () => {
  const commands = new Commands(() => testGitlabDeployment(), {}, new AbortController().signal)
  vi.spyOn(commands, 'gitlabPages').mockResolvedValue([])
  vi.spyOn(commands, 'gitlab').mockResolvedValue({ approved: false, approved_by: [] })
  expect(await readGitlab(commands, 'merge_requests/3/approvals')).toEqual({ approved: false, approved_by: [] })
})
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `pnpm vitest run tests/gitlab-read.spec.ts`
Expected: FAIL, module not found.

- [ ] **Step 3: Implement `gitlab-read.ts`; add `threadContext`/`find` to the kind; add the resource route**

`resources.ts`: add `{ name: 'gitlab/…', holds: 'a GitLab original, as the dsh-superagent-gitlab skill describes', executor: true }`, route `['gitlab/', read => sourcePage(read, 'gitlab', () => readGitlab(read.host.commands.withSignal(read.signal), read.resource))]`, and change the offset error to pick the label by source (`Feishu`/`GitHub`/`GitLab`).

- [ ] **Step 4: Run tests and typecheck**

Run: `pnpm vitest run tests/gitlab-read.spec.ts tests/tools.spec.ts` and `pnpm typecheck`
Expected: PASS.

- [ ] **Step 5: Update 007 and commit**

```bash
git add src/sources/gitlab-read.ts src/sources/gitlab.ts src/app/resources.ts tests/gitlab-read.spec.ts docs/007-architecture.md
git commit -m "feat: read GitLab threads and resources"
```

---

### Task 8: MR binding, Ticket end, and the Executor's report target

**Files:**
- Create: `src/sources/gitlab-mr.ts`
- Create: `src/sources/end-notices.ts`
- Modify: `src/app/runtime.ts` (end-notice dispatcher; `gitlab` SourceContext on `toolHost`)
- Modify: `src/app/tools/executor.ts` (`ExecutorToolsHost.gitlab`, `pr_number` dispatch, neutral descriptions)
- Test: `tests/gitlab.spec.ts` (append; define local `openTicket`, `execution` and `mergeOf(over)` fixtures from the Merge shape, patterned on `tests/sources.spec.ts`); `tests/tools.spec.ts` for the dispatch
- Modify: `docs/007-architecture.md` (Modules rows)

**Interfaces:**
- Consumes: `SourceContext`, `Ticket`, `OutboundMessage`, `deployment.gitlab`, GitHub's `ticketBranch`.
- Produces:
  ```ts
  export const MergeSchema: z.ZodType<Merge>
  export type Merge
  export const readMerge: (ctx: SourceContext, iid: number, signal?: AbortSignal) => Promise<Merge>
  export const mergesOf: (ticket: Pick<Ticket, 'follows'>, open?: boolean) => number[]
  export const agentMerge: (mr: Merge | { author: { username: string }; source_project_id?: number }, deployment: Pick<Deployment, 'gitlab'>) => boolean
  export const claimMerge: (ctx: SourceContext, ticketId: string, mr: Merge) => string | undefined
  export const observeMerge: (ctx: SourceContext, mr: Merge) => Promise<boolean>
  export function mergeEndNotices(ticket: Ticket, request: string, deployment: Deployment): OutboundMessage[]
  // src/sources/end-notices.ts
  export function changeEndNotices(ticket: Ticket, request: string, deployment: Deployment): OutboundMessage[]
  ```
  Rules: an MR belongs to a Ticket when it is the Ticket's origin MR or an MR by the deployment account whose `source_branch` last segment is the Ticket's `DSH_SUPERAGENT_NAME`; one MR binds to at most one open Ticket; merged/closed MRs call `ctx.tickets.threadClosed({ source: 'gitlab', id: 'mr:<iid>' }, mr.state === 'merged' ? 'merged' : 'closed', at)`. Notices: GitLab Channels are skipped, other Channels get `[MR !3](url) 已合并。`/`[MR !3](url) 已关闭，未合并。` (English when the request has no Han characters), message id `key('mr-terminal', String(iid), channel.source, JSON.stringify(channel.address), outcome)`.

- [ ] **Step 1: Write the failing tests**

```ts
it('binds an agent MR on the ticket branch and ends the ticket on merge', async () => {
  const h = host({ deployment: testGitlabDeployment() })
  const ticket = { ...openTicket('t1'), execution: execution('fix-a'), follows: [] }
  const mr = mergeOf({ iid: 3, source_branch: 'agent/fix-a', state: 'opened', author: { id: 7, username: 'agent-bot' }, source_project_id: 413 })
  expect(claimMerge(h, ticket.id, mr)).toBeUndefined()
  expect(h.tickets.get(ticket.id)!.follows).toEqual([{ thread: 'gitlab:mr:3', own: true }])
  expect(await observeMerge(h, mergeOf({ ...mr, state: 'merged', merged_at: '2026-11-19T03:00:00Z' }))).toBe(true)
  expect(h.tickets.get(ticket.id)!.ended).toMatchObject({ outcome: 'merged' })
})
it('builds the GitLab chat notice only for non-GitLab channels', () => {
  const ticket = { ...openTicket('t1'), ended: { outcome: 'merged', at: '2026-11-19T03:00:00Z' } as never,
    follows: [{ thread: 'gitlab:mr:3', own: true, closed: { outcome: 'merged', at: '2026-11-19T03:00:00Z' } }],
    channels: [{ source: 'feishu', address: { chat: 'oc_1', message: 'om_1' } }, { source: 'gitlab', address: { kind: 'mr', iid: '3' } }] }
  expect(mergeEndNotices(ticket, '修一下', testDeployment({ github: undefined, gitlab: testGitlabAccount() }))).toEqual([
    expect.objectContaining({ text: expect.stringContaining('已合并'), channel: { source: 'feishu', address: { chat: 'oc_1', message: 'om_1' } } })])
})
```

Also add to `tests/tools.spec.ts` (or `tests/gitlab.spec.ts`): `orchestrator_report` with `pr_number` on a Ticket whose origin Trigger is GitLab calls `readMerge`/`claimMerge`; the same on a GitHub Ticket still calls `claimPull`.

- [ ] **Step 2: Run tests to verify they fail**

Run: `pnpm vitest run tests/gitlab.spec.ts -t 'agent MR'`
Expected: FAIL, module not found.

- [ ] **Step 3: Implement `gitlab-mr.ts`, `end-notices.ts` and the wiring**

- `gitlab-mr.ts` mirrors `github-pull.ts`; import `ticketBranch` from `./github-pull.ts` (it is forge-neutral) instead of copying it.
- `end-notices.ts` switches on `follow.thread.split(':')[0]` to call `pullEndNotices` or `mergeEndNotices`; `runtime.ts` calls it where it called `pullEndNotices` (import from `src/sources/end-notices.ts`).
- `ExecutorToolsHost` gains `gitlab: SourceContext`; `runtime.ts` `toolHost` adds `get gitlab() { return runtime.context({ kind: 'gitlab', health: ['gitlab'] }) }`. In `executor.ts`, resolve the Ticket's origin source from `host.store.triggers.get(ticket.origin.triggerId)?.source` and dispatch `pr_number` to `readPull`/`readMerge`. Tool description: "The ticket's PR or MR number: its source PR/MR, or one by the repository account from a branch named after this ticket (DSH_SUPERAGENT_NAME)."

- [ ] **Step 4: Run tests and typecheck**

Run: `pnpm vitest run tests/gitlab.spec.ts tests/sources.spec.ts tests/tools.spec.ts` and `pnpm typecheck`
Expected: PASS; GitHub behaviour unchanged.

- [ ] **Step 5: Update 007 and commit**

```bash
git add src/sources/gitlab-mr.ts src/sources/end-notices.ts src/app/runtime.ts src/app/tools/executor.ts tests/gitlab.spec.ts tests/tools.spec.ts docs/007-architecture.md
git commit -m "feat: bind GitLab merge requests to tickets"
```

---

### Task 9: Polling loops

**Files:**
- Modify: `src/sources/gitlab.ts` (round, streams, notes, approvals, subscribed threads, ticket MRs, plugin loop)
- Modify: `src/sources/poll.ts` (move `pollFailure` here) and `src/sources/github.ts` (re-export it so existing imports keep working)
- Test: `tests/gitlab.spec.ts` (append)

**Interfaces:**
- Consumes: Tasks 4–8, `KV.gitlab`, `cursorAt`, `pollThreads`, `Tally`, `validItems`.
- Produces:
  ```ts
  export async function pollRound(ctx: SourceContext, signal?: AbortSignal): Promise<void>
  export async function notesSince(commands: Commands, path: string, since: string, tally: Tally): Promise<GitlabNote[]>
  export function approvalFingerprint(approvedBy: readonly string[]): string // sorted usernames joined with '\n'
  export async function pollThreads(ctx: SourceContext, signal?: AbortSignal, round?: Round): Promise<void>
  export async function pollTicketMerges(ctx: SourceContext, signal?: AbortSignal, round?: Round): Promise<void>
  ```
  Round semantics: streams read `?updated_after=<since>&order_by=updated_at&sort=asc&per_page=100` for Issues and MRs; per item: created Trigger (fresh only, not the agent's), state Trigger on `KV.gitlab.state` change, notes via `notesSince` (newest-first pages until older than `since`, cap 10 pages; cap exhausted throws `Error('GitLab note catch-up exceeded 10 pages for <thread>')`), approvals on fingerprint change; then `ctx.state.set(KV.gitlab.stream(kind), { since: next })`. `notesSince` returns oldest-first. Unaddressed notes are received only when `observeComments` is on or the Thread has subscriptions; addressed notes always. `pollThreads` covers subscribed Threads and MR state; `pollTicketMerges` reconciles open Tickets' MRs. One failure per item is recorded, not fatal; `failuresBeforeError` consecutive failed rounds report `health('error','pollFailures',{count,error})`, a success reports `ready` with counts.

- [ ] **Step 1: Write the failing tests**

Build the `host()` helper by copying the pattern from `tests/sources.spec.ts` (Store, registered kinds, Engine with stand-ins, `sourceContextOf`, `testGitlabDeployment()`), and a `glab(h, routes)` stub that spies on `h.commands.run`, keys on the project-relative path from `args[3]`, and returns the JSON for that path. Import `pollThreads` from `./poll.ts` as `pollEach` so the exported GitLab function can keep its name.

```ts
it('receives a new Issue comment once and advances the notes cursor', async () => {
  const h = host({ deployment: testGitlabDeployment() })
  glab(h, {
    'projects/group%2Fwidgets/issues?updated_after=': [issueFixture({ iid: 12, updated_at: '2026-11-19T01:10:00Z' })],
    'projects/group%2Fwidgets/merge_requests?updated_after=': [],
    'projects/group%2Fwidgets/issues/12/notes': [noteFixture({ id: 5, body: '@agent-bot please fix' })],
  })
  await pollRound(h)
  expect(h.receive).toHaveBeenCalledTimes(1)
  await pollRound(h)
  expect(h.receive).toHaveBeenCalledTimes(1) // duplicate id, same note
  expect(h.store.get(KV.gitlab.cursor('gitlab:issue:12'))).toBeDefined()
})
it('keeps cursors and receives nothing twice when a round is aborted mid-way', async () => {
  const h = host({ deployment: testGitlabDeployment() })
  const controller = new AbortController()
  glab(h, { /* first note received, second call aborts the controller */ })
  await expect(pollRound(h, controller.signal)).rejects.toThrow()
  expect(h.store.get(KV.gitlab.stream('issues'))).toBeUndefined()
})
it('fails instead of dropping notes past the catch-up cap', async () => {
  const h = host({ deployment: testGitlabDeployment() })
  glab(h, { 'issues/12/notes': Array.from({ length: 100 }, (_, i) => noteFixture({ id: 1000 - i })) }) // 10 full pages
  await expect(notesSince(h.commands, 'issues/12/notes', '2026-01-01T00:00:00Z', { skipped: 0 })).rejects.toThrow('catch-up exceeded')
})
it('reports an approval change once with a new revision', async () => {
  // approved_by [] then ['alice'] across two rounds -> one approval Trigger, revision 1
})
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `pnpm vitest run tests/gitlab.spec.ts -t 'new Issue comment'`
Expected: FAIL, `pollRound is not a function`.

- [ ] **Step 3: Implement the round and the plugin loop**

Add `watch` to `apply` (same shape as `github.ts`: wait `ctx.ready`, report `setupPending` while `!ctx.configured()`, loop with `pause`, abort on disposal). Report through `ctx.report('gitlab', …)`.

- [ ] **Step 4: Run tests and typecheck**

Run: `pnpm vitest run tests/gitlab.spec.ts tests/sources.spec.ts` and `pnpm typecheck`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/sources/gitlab.ts src/sources/poll.ts src/sources/github.ts tests/gitlab.spec.ts
git commit -m "feat: poll GitLab for Issues, MRs, notes and approvals"
```

---

### Task 10: Build and bundle registration

**Files:**
- Modify: `tsdown.config.ts` (`'gitlab-watch': 'src/sources/gitlab.ts'`)
- Modify: `package.json` (`"./gitlab-watch": "./lib/gitlab-watch.js"`; `files` already includes `lib`)
- Modify: `cordis.patch.yml` (insert `dsh-superagent-gitlab`, `disabled: true`)
- Modify: `examples/profile.cordis.patch.yml` (GitLab alternative comment)
- Test: `tests/bundle-patch.spec.ts` (expected row ids)

**Interfaces:**
- Consumes: the built plugin entry from Task 9 (`export const name`, `inject`, `apply`).
- Produces: `lib/gitlab-watch.js` and the bundle row.

- [ ] **Step 1: Write the failing test**

In `tests/bundle-patch.spec.ts` change the expected row ids to `['dsh-superagent', 'dsh-superagent-github', 'dsh-superagent-gitlab', 'dsh-superagent-feishu', …]`.

- [ ] **Step 2: Run the test to verify it fails**

Run: `pnpm vitest run tests/bundle-patch.spec.ts`
Expected: FAIL, row missing.

- [ ] **Step 3: Add the row, the entry and the export**

```yaml
    # Optional GitLab Source; requires a glab wrapper and a project after enabling.
    - id: dsh-superagent-gitlab
      name: dsh-superagent/gitlab-watch
      disabled: true
```

- [ ] **Step 4: Build and run the full check**

Run: `pnpm build` then `pnpm check`
Expected: `lib/gitlab-watch.js` exists; typecheck, tests and build pass.

- [ ] **Step 5: Commit**

```bash
git add tsdown.config.ts package.json cordis.patch.yml examples/profile.cordis.patch.yml tests/bundle-patch.spec.ts
git commit -m "feat: ship the GitLab Source in the bundle"
```

---

### Task 11: Prompts, skills and source-neutral wording

**Files:**
- Create: skills/dsh-superagent-gitlab/SKILL.md
- Modify: src/app/skill-files.ts (bundledSkills), src/app/context.ts (sourceSkills), src/app/prompts.ts (PartKey, builtInParts, composeSkill, Prompts constructor and skills()), src/app/runtime.ts (constructor argument)
- Modify: prompts/cli.md, prompts/orchestrator.md, prompts/worker.md, prompts/work.md, prompts/replies.md, prompts/dispatch.md, prompts/handoff.md, prompts/handoff-worker.md, prompts/team/communication.md, prompts/team/delivery.md
- Modify: tests/prompts.spec.ts, tests/prompt-config.spec.ts, tests/migration.spec.ts (the Prompts constructor's last argument), tests/cards.spec.ts, tests/model-surface.spec.ts

**Interfaces:**
- Consumes: the resolved Deployment; Task 1's optional forge fields.
- Produces:
~~~ts
type PartKey = ... | 'skill/gitlab'
export class Prompts { constructor(store: Store, maxChars: () => number, environment: () => Variable[] | undefined,
  sources: () => readonly ('github' | 'gitlab' | 'feishu')[]) }
skills(config: PromptConfig): [string, string][] // built-ins for configured Sources, then team skills
~~~
  Skill text: front matter name: dsh-superagent-gitlab, description: Read GitLab issues, merge requests, reviews and approvals using glab.; the body documents the gitlab/… resources, reply commands (glab api POST to projects/:id/merge_requests/:iid/discussions/:discussion_id/notes for a discussion, .../notes otherwise), that approvals are review signals delivered to the bound Ticket, and that CI is not delivered yet.

- [ ] **Step 1: Write the failing tests**

- tests/prompts.spec.ts: new Prompts(store, () => 100000, variables, () => ['github']).skills(config) lists github but not gitlab; with ['gitlab'] the reverse; with ['gitlab', 'feishu'] includes feishu.
- tests/cards.spec.ts: update the asserted sentences to the neutral wording in Step 3.
- tests/model-surface.spec.ts: register gitlabKind where the GitHub kind is registered and allow the measured prompt/skill budget to grow by at most 5%.

- [ ] **Step 2: Run tests to verify they fail**

Run: pnpm vitest run tests/prompts.spec.ts tests/cards.spec.ts tests/model-surface.spec.ts
Expected: FAIL on the new skill/part and the changed sentences.

- [ ] **Step 3: Implement the prompt and skill changes**

Neutral replacements (keep each sentence's meaning; do not add commands):
- cli.md line 6: use the Source's CLI for every repository call, gh on GitHub and glab on GitLab; specify --repo or an explicit API path; the matching dsh-superagent skill names the commands.
- orchestrator.md 14: act only as the repository account the Execution environment names; 15: never merge or enable auto-merge yourself, close issues or expose private Feishu content on the repository.
- worker.md 5: use the source CLI (gh or glab) for ALL repository operations, and commit as the author the Delivery requirements section names, otherwise as the account and commit email the Execution environment names; 6: never merge a PR/MR directly or enable auto-merge other than as the Delivery requirements section says, close issues or expose private Feishu content on the repository; 8: load the matching dsh-superagent skill for Feishu or the repository forge.
- work.md 10, 11, 17, 19, 22, 48: GitHub/PR become the repository/PR or MR; keep DSH_SUPERAGENT_NAME and pr_number.
- replies.md 26, 34, 35; dispatch.md 21, 24; handoff.md 3; handoff-worker.md 2: forge-neutral wording; repository comments on GitHub and GitLab support Markdown; never copy private material onto the repository.
- team/communication.md 3, 55, 68, 83: repository-forge wording; the disclosure line applies to both forges; bots are GitHub Apps and [bot] logins, and GitLab bot users.
- team/delivery.md 10, 15, 17: the auto-merge example names both CLIs (glab mr merge <iid> --auto-merge and the gh form); the co-author id lookup is gh api users/<login> or glab api 'users?username=<login>'; the default stays a person merges because CI is out of scope.

Also: add the gitlab skill to bundledSkills/sourceSkills; add the skill/gitlab part to PartKey/builtInParts/composeSkill; change Prompts's last constructor argument from feishu: () => boolean to the configured-kinds function, and pass from runtime.ts: () => [...setup.deployment.github ? ['github'] : [], ...setup.deployment.gitlab ? ['gitlab'] : [], ...feishu() ? ['feishu'] : []].

- [ ] **Step 4: Run tests and typecheck**

Run: pnpm vitest run tests/prompts.spec.ts tests/cards.spec.ts tests/model-surface.spec.ts tests/prompt-config.spec.ts tests/migration.spec.ts and pnpm typecheck
Expected: PASS.

- [ ] **Step 5: Commit**

~~~bash
git add skills/dsh-superagent-gitlab src/app/skill-files.ts src/app/context.ts src/app/prompts.ts src/app/runtime.ts prompts tests/prompts.spec.ts tests/cards.spec.ts tests/model-surface.spec.ts
git commit -m "feat: add the GitLab skill and forge-neutral prompts"
~~~

---

### Task 12: Panel copy and localization

**Files:**
- Modify: src/client/locales.ts (both languages: source.gitlab, health.gitlab, promptPart.skill/gitlab, problemSetupFix.gitlab, every new setup.* code, and the GitHub-only copy below)
- Modify: tests/locales.spec.ts, tests/panel-deployment.spec.tsx, tests/panel-prompts.spec.tsx, tests/http.spec.ts only where they assert the changed shape
- Test: tests/locales.spec.ts (key parity) and the panel tests

**Interfaces:**
- Consumes: Task 1's setupView().accounts (forge, agent) and setup codes; Task 11's skill/gitlab part.
- Produces: both locale files with identical key sets; no user-visible copy says GitHub where the panel can show GitLab.

- [ ] **Step 1: Write the failing test**

Extend tests/locales.spec.ts: assert setup.projectUnset, setup.gitlabCliUnset, setup.repositoryAmbiguous, source.gitlab, health.gitlab, promptPart.skill/gitlab and problemSetupFix.gitlab exist in both language maps.

- [ ] **Step 2: Run the test to verify it fails**

Run: pnpm vitest run tests/locales.spec.ts
Expected: FAIL, missing keys.

- [ ] **Step 3: Add the copy in both languages**

Chinese/English pairs for source.gitlab, health.gitlab, promptPart.skill/gitlab, problemSetupFix.gitlab, all nine setup.* codes; update setup.gitTooOld, searchNoneGithub, meaning.agent and promptPartHint.communication to name both forges. The Deployment page keeps its Repository/Agent rows, so no component change is needed; update a panel fixture only if it asserted the old accounts object.

- [ ] **Step 4: Run the panel and locale tests**

Run: pnpm vitest run tests/locales.spec.ts tests/panel-deployment.spec.tsx tests/panel-prompts.spec.tsx tests/http.spec.ts
Expected: PASS.

- [ ] **Step 5: Commit**

~~~bash
git add src/client/locales.ts tests/locales.spec.ts tests/panel-deployment.spec.tsx tests/panel-prompts.spec.tsx tests/http.spec.ts
git commit -m "feat: localize the GitLab Source and its setup problems"
~~~

---

### Task 13: Documentation 001–005 and README

**Files:**
- Modify: docs/001-domain.md, docs/002-authorization.md, docs/003-execution.md, docs/004-prompts.md, docs/005-panel.md, README.md
- Test: tests/docs.spec.ts

- [ ] **Step 1: Confirm the docs test passes before editing**

Run: pnpm vitest run tests/docs.spec.ts
Expected: PASS.

- [ ] **Step 2: Edit the documents**

001: add GitLab to the Source rows of What is reusable and to the Agent glossary line. 002: add GitLab's admission rows (issues, merges, comments) beside GitHub's; approvals are subscription-only. 003: add the DSH_SUPERAGENT_GITLAB_* variables where the GitHub ones are listed. 004: list skill/gitlab beside skill/github and skill/feishu. 005: the Sources/Deployment pages show the GitLab card and account. 007: update the responsibility lines of src/deployment.ts, src/infra/commands.ts, src/infra/environment.ts, src/app/resources.ts and src/app/prompts.ts. README: one line that the Source speaks GitHub or GitLab.

- [ ] **Step 3: Re-run docs and links**

Run: pnpm vitest run tests/docs.spec.ts
Expected: PASS.

- [ ] **Step 4: Commit**

~~~bash
git add docs/001-domain.md docs/002-authorization.md docs/003-execution.md docs/004-prompts.md docs/005-panel.md README.md
git commit -m "docs: describe the GitLab Source"
~~~

---

### Task 14: Documentation 006 and SETUP.md for a GitLab installation

**Files:**
- Modify: docs/006-deployment.md (Prerequisites, Accounts, Install, Machine configuration, Settings, Management API, Setup checks, Try it, Troubleshooting)
- Modify: SETUP.md (the guided flow asks for GitLab)
- Test: tests/docs.spec.ts

- [ ] **Step 1: Edit 006**

Add a glab 1.124+ prerequisite; an Agent GitLab account section with the glab-agent wrapper (GLAB_CONFIG_DIR="$HOME/.config/glab-agent" exec glab "$@", glab-agent auth login --hostname <host>, glab-agent api --hostname <host> user); the machine-config note that the GitLab profile enables dsh-superagent-gitlab and disables dsh-superagent-github; the Settings source:gitlab block (project, host, cli, observe switches) and policy.sources.gitlab.allowed; a GitLab prepare command (git -C <clone> fetch --no-tags https://$DSH_SUPERAGENT_GITLAB_HOST/$DSH_SUPERAGENT_GITLAB_PROJECT.git HEAD && git -C <clone> worktree add -b agent/$DSH_SUPERAGENT_NAME $DSH_SUPERAGENT_DIR FETCH_HEAD); Setup checks (the glab shim and the credential.https://<host> helper); Try it with #<iid> Issues and !<iid> MRs.

- [ ] **Step 2: Edit SETUP.md**

The Sources question becomes GitHub or GitLab? (one work repository per installation), with GitLab recommended for this operator; the Repository question becomes group/project; the Agent account is a GitLab user; the allowed list is GitLab usernames; the checkout and prepare use the GitLab URL; commit identity uses the agent's commit_email; the auto-merge answer stays a person merges. The operator-steps block uses the wrapper and membership commands from 006.

- [ ] **Step 3: Run the docs test**

Run: pnpm vitest run tests/docs.spec.ts
Expected: PASS.

- [ ] **Step 4: Commit**

~~~bash
git add docs/006-deployment.md SETUP.md
git commit -m "docs: guide a GitLab installation"
~~~

---

### Task 15: Install and verify on this machine

**Files:** none in the repo unless the check finds a gap, in which case the file that fixes it.

- [ ] **Step 1: Run the full check on the branch**

Run: pnpm check; confirm dsh --version (0.2.2-rc.1 on this machine) satisfies engines.dsh in package.json. Expected: typecheck, all tests and build pass.

- [ ] **Step 2: Collect the operator answers, one question per message**

Target GitLab project (group/project), the GitLab account the agent acts as (a dedicated account is recommended), the GitLab users who may ask for work, and the checkout path for Tickets.

- [ ] **Step 3: Operator steps in one message**

Create or choose the agent's GitLab user and add it to the project as Developer+; create ~/.local/bin/glab-agent with GLAB_CONFIG_DIR="$HOME/.config/glab-agent"; run glab-agent auth login --hostname gitlab.xiding.tech (the token never appears in chat); confirm glab-agent api --hostname gitlab.xiding.tech user. Wait for done, then check each login.

- [ ] **Step 4: Create the profile and machine configuration**

~~~bash
dsh superagent --from-default-profile web --dump-config > /dev/null
dsh plugin --profile superagent add "link:$PWD"
~~~
Write ~/.dsh/profiles/superagent/cordis.patch.yml: port 3082, stateDir ~/.dsh/state/superagent, dsh-superagent-gitlab with disabled: false, dsh-superagent-github with disabled: true.

- [ ] **Step 5: Start and save settings through the Management API**

~~~bash
scripts/dsh-superagent start
api=http://127.0.0.1:3082/orchestrator/api
# read the revision from GET settings, then PUT settings/source:gitlab, settings/policy,
# settings/executor (directory, prepare command, blockedCommands), and prompts (Delivery requirements).
~~~
Each save returns restarting: true; wait until scripts/dsh-superagent status exits 0 before the next request.

- [ ] **Step 6: Verify end to end**

scripts/dsh-superagent status exits 0; GET sources shows gitlab ready; as an allowed user comment @<agent> <request> on a test Issue; within about a minute a Ticket appears, the agent acknowledges and replies on the Issue; from the ticket branch it opens an MR; an approval and a review comment reach the worker; merging the MR ends the Ticket. Record what was verified in the handoff message.

- [ ] **Step 7: Commit any fixes and hand over**

~~~bash
git add -A && git commit -m "fix: what the install check found"   # only if needed
~~~
Send the operator: the panel URL, how to try it, the three responsibility rules from docs/000, and where to read more.
