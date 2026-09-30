# Main loop: specification

Status: **DRAFT v0.5.1**, 2026-09-28. Not normative. Repository `curator-keeper`. Track: [`track.md`](track.md). Diagrams: [`../diagrams/`](../diagrams/). Earlier research: wiki `research/keeper-and-process-lifecycle.md` and `research/agent-loop.md`; roadmap line KP (`wiki/roadmap/ecosystem-roadmap.md` §5). Command-line shape: wiki `research/cli-shaping-review.md`, adopted 2026-09-27. Change logs: §16 (v0 → v0.1), §17 (v0.1 → v0.2), §18 (v0.2 → v0.3), §19 (v0.3 → v0.4), §20 (v0.4 → v0.5) and §21 (v0.5 → v0.5.1).

One person runs several processes on one machine: a family calendar, the smart home, a hobby project, a small café. Each process has its own playbook, board and root orchestrator (curator-playbook `spec/process-configuration.md`). Something has to sit above them. It must:

- know which processes exist;
- take a request in plain words, such as "order oat milk", and decide which process it belongs to;
- start that process's orchestrator when it is not running, and wake it when a message arrives;
- bring results back to the person;
- ask the person what only a person may decide;
- bring the whole machine back after a reboot.

This specification describes that component, the **main loop**: precisely for one machine, and for several machines as far as the architecture must allow now.

## Names *(decided 2026-09-25)*

People see the beekeeping names; this specification's prose uses the technical terms. The map below applies wherever a name first appears.

Beekeeping words appear only where people type them in the apiary: the `keeper` command and its nouns (`keeper hive …`), the apiary's directories (`~/apiary`, `hives/`) and the profile name `apiary`. Every kernel command, flag, configuration key and provider uses the technical terms, so `keeper.toml` lists `[processes.*]`, and task-board, the session host and Curator say process, session and runtime. The same rule stands in the session-host specification (Names) and in curator-playbook §16. *decided 2026-09-27*

| Name | Technical term | What it is |
| --- | --- | --- |
| **apiary** | the whole system | every yard, hive and operator of one owner; one machine is a small apiary |
| **yard** | node | one machine of the apiary |
| **keeper** | main loop | the master orchestrator of a yard: one per node, and one head keeper when an apiary has several yards |
| **hive** | process | one domain: family calendar, smart home, café, a hobby project |
| **queen** | root orchestrator | exactly one per hive, started from outside the hive |
| **comb** | board | its cells are the tasks |
| **bee** | run | an agent at work, sub-orchestrators included |
| **wild bees** | external participants | agents of another owner working on one of our hives under leases and signatures |
| **nuc** | process package | the published playbook a new hive grows from |

Markers:
- *decided (date)*: the operator decided it;
- *proposed*: the author's proposal;
- **pending operator decision**: named where it applies.

Related documents:
- curator-playbook `spec/process-configuration.md`, DRAFT v4.4 (the core): processes, root orchestrators, hand-off, delegation semantics, skill policy; a version of a process takes effect when the trust component reports it confirmed;
- relux-works/curator-trust `spec/trust.md`: who counts as a person, confirmations and their batching, capability grants, yolo mode, the companion app and pairing, secrets, and separation at the operating-system level;
- waggle `spec/waggle.md` with Amendment A1: messages, doorbell, carriers, leases, board acts;
- the session-host specification (`agent-session-host`);
- curator-spec Decisions 0010 (environment profiles), 0013 and 0019 (launch plans, one construction site) and 0021 (sessions enter through `curator run`, hosted through the `ax` contract); curator-spec issues #92 (source shorthands and default directories), #93 (`profile export` and `profile restore` of profile declarations), #94 (attestation fast path and signed bootstrap), #95 (command-only installs) and #100 (provider installs); curator issues #87 (the Go toolchain policy) and #96 (provider installs);
- the command-line shape of `keeper`, `curator` and `task-board`: wiki `research/cli-shaping-review.md`, adopted 2026-09-27;
- the agent-session-manager (`ax`) specification;
- curator-inference-manager `spec/inference-plane.md`.

---

## 0. The design on one screen

- **One main loop per node.** It manages the node's **processes** — each a project directory with its own `Playbook.json`, board and exactly one **root orchestrator** — and does no process work itself. *proposed; one root orchestrator per process, started from outside, decided (2026-09-24)*
- **The session host keeps orchestrators alive.** The main loop starts a process's root orchestrator through the session host, which keeps it running in the background. The root orchestrator runs its own process and spawns sub-orchestrators and runs inside it. *decided (2026-09-24)*
- **The main loop and orchestrators talk over waggle.** A request goes in as a message, the session host rings the doorbell, and the orchestrator pulls the message; reports come back the same way. Child runs never talk to the main loop. *proposed*
- **A registry points at separate repositories.** The main loop keeps a **process registry**, `keeper.toml`, in its own **root directory**. Processes are separate repositories the registry points at, never git submodules. Adding, removing or moving a process is authority-bearing and takes effect after a person confirms it. *proposed*
- **Installation is detached from us, and fast.** The root is instantiated from a pinned, audited template package and becomes the person's own repository with no upstream to us. Our code reaches a node only through the Curator registry, audited and pinned. The installer uses the system's git and Go when they fit and never installs the full Xcode; a signed bootstrap and signed registry attestations replace a long local audit. *decided (2026-09-25)*
- **One install brings everything.** The keeper package's `apiary` profile is the install unit: one `curator profile install relux-works/curator-playbook-keeper` brings the main loop's skill, the board tool, session-host tooling and the memory skill. `keeper init` then adds the system roles' profiles and the Curator providers the node needs, which no profile can carry (§6.2). One tool serves many boards. *proposed; the operator's direction (2026-09-25)*
- **`keeper` opens the conversation.** With no arguments it opens the conversation with the main loop, running `keeper up` first when the apiary is down. `keeper chat <hive>` talks with a hive's queen; `keeper watch <target>` only looks. *proposed; the operator's direction (2026-09-25)*
- **`keeper up` is the single entry point.** It brings a node up — at login, after a reboot, or by hand — and resumes every hive from its goal and park note. `keeper down` parks everything; `keeper estop` stops everything at once. *proposed*
- **Routing** follows a fixed order *(decided 2026-09-24)*:
  1. an explicit reference wins;
  2. then a channel bound to a process;
  3. then one confident candidate;
  4. close candidates get one short question, and the answer becomes a visible routing hint;
  5. no candidate leads to the process designer;
  6. a request for two processes is split.

  A fast classifier with typed outputs fits; the specification does not prescribe one.
- **The main loop is a role plus a module.** The role is a skill any harness can run; the module is the `keeper` command and library that does the deterministic work. The minimal system runs entirely on existing harnesses. Our own lightweight loop is optional and later. *proposed*
- **Three front doors, by audience.** `keeper` serves the person in an apiary. `curator` installs and configures the machine, and dispatches a noun it does not know to that noun's provider, `curator-<noun>` (`curator run`, `curator session`, `curator models`, …). `task-board` is the kernel that agents, the session host, the timer and `keeper` call. keeper's commands wrap the other two, and one verb keeps one meaning across all three (§7.2). *decided 2026-09-27*
- **Processes are created, not configured by hand.** The main loop matches a request against published templates first. When none fits, the **process designer** generates a playbook, always starting from the simplest process that works. The **librarian**, a small process of its own on a timer, proposes merging near-duplicates and merges only after the person approves. *decided (2026-09-24)*
- **System roles keep the node healthy.** The **sysadmin** backs the apiary up to the person's own remotes, restores it, keeps an inventory of tokens and updates skills. The **watchdog** watches context and provider limits and parks sessions before they run out. *decided in direction (2026-09-25); protocols proposed*
- **The person talks and approves remotely.** Conversation goes through channels: the terminal first, then messengers through waggle providers. Approvals go through a **companion app** — on the phone in the common case, or on another computer — or through the operating system's user authentication on the machine. *decided (2026-09-24)*
- **Permissions work like a phone's.** A hive asks once for a kind of change, such as creating tasks on a schedule, and the person answers always, once or no. Changes to a process's structure wait for one batched touch, and a hive or the whole apiary may run in yolo mode (curator-trust). *decided in direction (2026-09-25)*
- **Memory is a plug-and-play skill.** Notes are facts with provenance, never instructions, written under a daily budget; they live in the apiary's repositories and are backed up with them (§13.4). *proposed*
- **Several nodes are architected for, not implemented now:** one process worked from several nodes under leases, unrelated processes on several nodes under a head main loop, several operators, and external participants. *proposed*

## 1. Goals, non-goals and principles

Goals:

1. A person reaches every process on the node through one conversation, without knowing how the processes are organized.
2. A new kind of work becomes a new process without hand-editing configuration.
3. Background orchestrators run autonomously, survive restarts and reboots, and are observable on demand.
4. What only a person may decide reaches the person wherever they are, and the answer carries proof that the person gave it.
5. Nothing in the one-node design blocks several nodes, several operators or external participants later.

Non-goals of this draft:
- the process model (curator-playbook);
- message formats, signatures and carriers (waggle);
- how sessions are hosted (the session-host specification);
- our own agent loop (wiki `research/agent-loop.md`);
- the companion app's implementation (§11.3 lists what the main loop needs from it).

### 1.1 Principles *(proposed)*

1. **The journal is the state; a session is a cache.** Everything a process needs to continue lives in its repository and on its board: the goal, park notes, decisions, runs, Change Requests. A session can be compacted, restarted, replaced by another harness or lost in a reboot without losing the process. Park notes (§10.5), restore (§10.6) and compaction hand-offs (§12.2) all follow from this.
2. **Authority only narrows.** Every nested or delegated authority is a subset of its parent's: operator, main loop, root orchestrator, sub-orchestrator, run. A delegation never exceeds what its grantor may do, and a sub-orchestrator's pool never exceeds its parent's.
3. **Transport is not authority.** A tailnet identity, a relay, or a messenger account shows only where a message came from. What it may do is decided by signatures, the roster and the process rules.
4. **Privileged operations happen outside agent shells.** Secrets — tokens, signing keys, credentials — never enter an agent's environment. Privileged operations are everything that spends money, acts outside the apiary on the person's behalf, or cannot be undone:
   - payments and transfers;
   - signing and publishing builds and packages;
   - deploying to production;
   - using secrets: API keys, tokens, passwords;
   - messages to people outside the apiary on the person's behalf;
   - acting on physical devices, such as a lock or an alarm;
   - deleting data or backups;
   - database migrations.

   An agent requests such an operation and never holds the means to perform it. A trusted executor — the kernel executor, the sysadmin's tooling or the companion app — performs it after authorization and, where the policy requires it, after a person's touch (curator-trust).
5. **The emergency stop belongs to the authority plane.** `keeper estop` is an operator act that needs a person present; the main loop may trigger it only under a delegation that names the trigger, and root orchestrators never. The session host and the kernel execute it; no model mediates it (§10.3).

## 2. Terms

| Term | Meaning |
| --- | --- |
| node | one machine with its operator's installation |
| main loop | the component this specification describes; one per node |
| process | one domain of work: a project directory with `Playbook.json`, `Skillfile.json`, `task-board.config.json` and a board (curator-playbook §2.1); a composition of several playbooks on one board counts as one process |
| root orchestrator | the single orchestrator session of a process that is started from outside it: the process's root role (curator-playbook §4.4) |
| sub-orchestrator, run | agents that a root orchestrator spawns inside its own process |
| process registry | the main loop's list of the node's processes (§5.2) |
| process package | a published playbook: a `curator-playbook-<purpose>` skill collection (curator-playbook §8) |
| keeper package | the one package the root installs: the main loop's orchestrator skill, its template and its dependencies (§6.2) |
| session host | the node's component that starts, keeps and wakes long-lived agent sessions (the session-host specification; today task-board's session daemon `tb-sessiond`); its command line is `curator session …`, which `keeper` fronts |
| kernel | task-board's own code, which keeps boards and runs deterministic steps without a model |
| operator | a person who administers the node and answers what only a person may decide |
| external participant | an agent of another owner that works on one of our processes under a lease and signatures |
| companion app | the operator's app that renders requests for approval and signs them (§11.3) |
| park note | what a session writes down when it stops, bound into its goal so that the next start reads it first (session-host specification) |

## 3. One node

### 3.1 Who does what

| Component | Does | Does not |
| --- | --- | --- |
| main loop | keeps the registry; takes the person's requests and routes them; creates processes through the designer; starts, wakes, parks and stops root orchestrators; relays results and questions; runs its own small process with the system roles (§5.1, §12) | write any process's board; spawn a run inside a process; change a process's `Playbook.json` except by a committed change the person confirms |
| session host | starts sessions in a given directory with a given profile and goal; keeps them alive while detached; lets people attach; rings the doorbell; emits events; enforces who may act on which session (§10.2) | decide what a session does |
| root orchestrator | runs its process: plans, spawns sub-orchestrators and runs, integrates, asks for people through the main loop | write another process's board |
| sub-orchestrators, runs | the work inside one process, under the root orchestrator's leases | talk to the main loop |
| operator | answers and approves; confirms authority-bearing changes; is the only one who may pull the emergency stop | edit configuration by hand (possible, never required) |

### 3.2 Invariants

1. **One root orchestrator per process, started from outside.** *decided (2026-09-24)*
   - The main loop starts only the root orchestrator of a process; the root orchestrator starts everything else inside the process.
   - Today's board enforces this already: it has one primary-goal owner. A second primary launch either reattaches the running session or, for another provider, is refused with `session_provider_conflict` (skill-project-management `docs/claude-goal-runtime.md`).
2. **The main loop does no process work.** It never writes a process's board and never spawns a run inside a process. It reaches a process only through the process's root orchestrator. *proposed*
3. **Processes are isolated.** An orchestrator writes only its own board, and inside it only its leased subtree. Another process is reached only by hand-off (curator-playbook §6.5, §7.2). *decided*
4. **The registry is authority-bearing** (§5.4). *proposed*
5. **What only a person may decide goes to a person.** It travels through the main loop, and the answer is recorded with proof wherever the process requires it (curator-playbook §4.2). *decided*

### 3.3 Example

One node carries five processes: a family calendar, the smart home, a hobby app, a small café, and the main loop's own process. The person writes: "move Saturday's dentist to 11 and order oat milk".

1. The main loop splits the request (§8.2).
2. It sends the first half to the calendar's root orchestrator and the second to the café's:
   - the café's orchestrator is not running, so the main loop starts it first;
   - the calendar's orchestrator is parked, so the main loop wakes it (§10).
3. A purchase above the café's delegated limit waits for the operator's approval on the phone (§11.2).
4. The main loop answers with both results.

![One node: the main loop and its processes](../diagrams/artefacts/plantuml/one-node-system.svg)

## 4. Several nodes *(proposed; architected for, not implemented)*

A system outgrows one node in four ways, and the design keeps each open:

1. **One process, several nodes.** One project is developed from two machines, each with its own main loop.
   - The process keeps one board and one root orchestrator, on the node that owns it.
   - Orchestrators on other nodes work leased subtrees and coordinate over waggle (waggle §12, milestone CM2).
   - The owning node's registry lists the process as owned. The other nodes list it as *joined*: they may start sub-orchestrators under a lease, never a second root orchestrator.
2. **Unrelated processes, one system.** A second machine carries its own processes, such as a home server with the smart home, and still belongs to the owner's system.
   - One main loop is the **head main loop**, on the owner's control node. The others report to it over waggle (`coord` requests and reports) and take the requests it routes.
   - A request the head routes to a process on another node travels as a waggle message to that node's main loop, which delivers it to the root orchestrator.
   - The relation is coordination, not command: each node's main loop still starts and stops its own root orchestrators.
3. **Several operators.** Several people administer one system, possibly in a hierarchy. Operators are waggle principals (waggle §3), and who may confirm what is policy (curator-playbook §2.3, §4.2). An operator hierarchy is an open question (§15).
4. **External participants.** Agents of another owner work on one of our processes under a lease and signatures, admitted by that process's roster (waggle milestone CM4).

Three choices now keep all four ways open without a redesign:
- every registry entry names the process's owning node and a stable process id (§5.2);
- every message between a main loop and an orchestrator is a waggle message, even on one node, so crossing machines changes only the carrier (§10.4);
- processes are separate repositories, so a process can move to another node or be cloned there (§5.3).

![Several nodes](../diagrams/artefacts/plantuml/several-nodes.svg)

## 5. The root directory

### 5.1 Layout *(proposed)*

```
~/apiary/                      the main loop's root: the person's own git repository, no upstream to us
  curator/profiles.json        the node's profile and overlay declarations with their pins, from `curator profile export` (§6.2; curator-spec #93)
  Playbook.json                the main loop's own process: requests, routing, "a person is needed", system roles
  keeper.toml                  the process registry (§5.2); no machine facts: the default harness is an operator binding (§7.4)
  task-board.config.json       where the main loop's board lives
  .task-board/                 the main loop's board: requests and what became of them
  persona/                     the operator overlay: the persona (§13.2)
  memory/                      the notes of the main loop and the system roles (§13.4)
  .gitignore                   ignores hives/
  hives/                       default place for process directories, each its own repository
    family-calendar/           Playbook.json, Skillfile.json, task-board.config.json, .task-board/, memory/
    cafe/
```

- **The main loop has a process of its own.** Its elements are the person's requests and the questions the main loop asks, and its workflows track each request from arrival to answer. The system roles run in it: the designer, the librarian on a schedule, the sysadmin and the watchdog (§9, §12). So the main loop's work is visible on a board like any other process's.
- **Process directories may live anywhere.** `~/work/relux-app` is as good as `~/apiary/hives/relux-app`; the registry records the path.
- **No project-scope skills by default.** The root has no `Skillfile.json` of its own: the main loop's skills come with its profile, and every role's skills with that role's profile (§13.5). A process may still carry one for project-scope skills; role skills never go there.

### 5.2 The process registry *(proposed)*

```toml
[processes.family-calendar]
id          = "proc-7f3a2c"                  # stable, never reused
path        = "hives/family-calendar"        # relative to the root, or absolute
remote      = "git@github.com:example/family-calendar.git"   # optional: the person's own remote (§12.1)
node        = "home-mac"                     # the owning node
role        = "calendar-orchestrator"        # the root orchestrator's role alias
description = "The family's appointments, school events and reminders"
channels    = ["telegram:family-chat"]       # conversations bound to this process

[processes.cafe]
id          = "proc-19be04"
path        = "hives/cafe"
node        = "home-mac"
role        = "cafe-orchestrator"
description = "The café: purchasing, stock, supplier orders, hand-off to bookkeeping"

[[hints]]
text    = "milk on weekdays, from the work phone"
process = "cafe"
source  = "the person's answer, 2026-09-24"
```

- `id` is stable and never reused. `node` names the owning node; a joined process (§4) says `joined = true`.
- `remote` is always the person's own remote (their GitHub, GitLab or any git server), used for backup and for cloning the process onto another node (§12.1). It never points at us.
- `description` is what routing reads, so it says what the process handles in the person's words. The designer writes it and the person may edit it.
- The registry holds declarations only. Which sessions run, and in which state, is runtime state the session host keeps, never committed.
- **No machine facts.** Which harness runs a hive is not a registry key: it is an operator binding in `~/.curator/models.toml` (§7.4). The registry is committed, and a restore copies it to machines where that harness may not be logged in (§12.1). *decided 2026-09-27*

### 5.3 Separate repositories, not submodules *(proposed)*

Each process is its own repository, or a plain directory when it does not use git (curator-playbook §2.4). The root only points at it, for five reasons:

- **A submodule pins a commit in the parent.** Every change a process lands would need a second commit in the root to move the pin, or the root would describe a stale process. The kernel reads a process's rules from the head of that process's own default branch (curator-playbook §2.4); a pinned submodule commit is exactly what that rule avoids.
- **A submodule sits on a detached HEAD by default.** Agents and tools that commit into it lose work or commit off-branch.
- **Processes differ in owner, visibility and hosting.** A family calendar stays private, a hobby app may be public, and a café's books may live with an accountant. A submodule ties them all to one parent's access.
- **A process may not use git at all.**
- **A process moves between nodes, or is cloned to another node, on its own** (§4).

The root ignores `hives/`, so the default location nests the processes on disk without making them part of the root repository.

### 5.4 Authority of the registry *(proposed)*

- **Which entries are authority-bearing.** These entries decide where requests and authority go:
  - adding, removing or moving a process;
  - changing a process's path, remote, owning node or root role;
  - binding a channel to a process.

  They follow the rule curator-playbook applies to a process's configuration. The main loop may write them, as a commit to its root repository, but they take effect only after a person confirms the new version (§11.2).
- **Which are not.** Descriptions and routing hints only choose where a request is offered; every process still applies its own rules. The main loop writes hints from the person's answers (§8.3), and they take effect at once.
- **The commands.** `keeper hive add`, `keeper hive remove` and `keeper hive set <hive>` write the entries; moving a process is `keeper hive set <hive> --path <dir>` (§7.2).

## 6. Installation from zero

### 6.1 Detached from us *(decided 2026-09-25)*

- **The root is instantiated, not cloned.** Curator copies a template package — pinned by its package identity and audited like any skill — into `~/apiary`, and the person's first commit starts the history. No remote pointing at us is added, and nothing is ever pulled from our repositories. So no repository of ours can push a change into a node.
- **Our code enters a node only through the Curator registry.** Every skill and command a node runs arrives as an audited package version pinned in the root's lock. The template, too, is such a package: a template update is a new version the person adopts, not a pull.
- **The main loop and the root orchestrators update their own skills.** When the registry publishes a new audited version of the keeper package or of a process's skills, the main loop (for the root) or a root orchestrator (for its process) proposes the update. It installs when the project's skill policy or a delegation allows it (curator-playbook §8.6), and the change it makes to a process waits for confirmation like any other (§9.6).

### 6.2 One install brings everything *(proposed; the operator's direction, 2026-09-25)*

The install unit is a profile. The keeper package ships context packages under `profiles/`: `apiary` for the main loop's session, and one profile for each system role. Curator already installs a profile from a directory of a repository, pinned by a tag, a range or a revision, and resolves the skills that the profile's own `Skillfile.json` requires (curator-spec Decision 0010; `protocol/environments.md` §1, §9.1). So one command installs the main loop:

```
curator profile install relux-works/curator-playbook-keeper
```

- **The short form** comes from curator-spec issue #92: `owner/repo` on the default host, and the default directory named by the package's index, here `profiles/apiary`. Until it ships, the full form is `curator profile install https://github.com/relux-works/curator-playbook-keeper --directory profiles/apiary`. Without a requirement flag Curator takes the latest version tag (`--range latest`).
- **The system roles' profiles.** `keeper init` then installs them from the same package at the same pinned version: `designer`, `librarian`, `sysadmin` and `watchdog`.

| The `apiary` profile requires | Mode | What it brings |
| --- | --- | --- |
| the keeper skill | `full` | the main loop's role, its process template and the `keeper` command |
| task-board | `runtime`, narrowed to the `task-board` command | the board tool: the kernel, spawns, the session daemon today |
| session-host tooling | `runtime`, narrowed to its commands | the session host, until it ships inside task-board's successor |
| the memory skill | `runtime`, narrowed to the memory package's command | notes with provenance (§13.4) |
| waggle tooling | `runtime`, once waggle CM1 ships | mail, doorbell, providers |

- **The `keeper` command on the person's `PATH`.** The profile's skills reach only the apiary's own sessions, but the person types `keeper` in any terminal. The installer therefore also adds the keeper skill globally with `curator global add … --commands-only`: commands only, no context. That publishes the `keeper` shim without materializing the main loop's role anywhere (curator-spec issue #95, curator issue #95; the lock may still record the activation as `runtime`). Until it ships, `keeper` reaches the person's `PATH` only while `apiary` is the machine-current profile, through Curator's forwarding of that profile's commands (curator-spec Decision 0016). *decided 2026-09-25: the keeper skill lives only in the `apiary` profile, and the `keeper` command is global (curator-spec #95, curator #95; the on-demand route, curator-spec #91 and curator #91, is closed)*

- **Curator's providers are installed, not carried by a profile.** `curator <noun>` dispatches to an executable named `curator-<noun>`, found only on Curator's trust roots, so no profile or skill can deliver one. The node needs `curator-run` for `keeper up` (§7.3) and `curator-session` for keeper's session commands (§7.2), and later `curator-models`, `curator-runtimes`, `curator-engines`, `curator-network` and `curator-playbook`. First-party providers ship beside `curator` in its signed release. `keeper init` runs `curator provider install <noun>` for each one the node lacks, `run` and `session` first, and `curator provider check` afterwards (curator-spec issue #100, curator issue #96). *decided 2026-09-27*
- **One version to update.** Every profile of the node pins the same keeper package version, so updating the package updates the set.
- **The declarations are committed.** The node's profile and overlay declarations, with their pins, are exported into the root's `curator/profiles.json` with `curator profile export`, and `curator profile restore` reads them back (§12.1; curator-spec issue #93, which names the restore side `restore` because `curator profile import` already means something else). The root repository therefore describes exactly what the node runs, and restore re-creates it.
- **One tool, many boards.** The `task-board` command is installed once per node: a Curator command shim has exactly one owner and one version on a node. Every process's board is data in that process's own repository (`.task-board/`). The keeper package and every process's orchestrator skill declare the board tool as a dependency so that Curator can check they agree; a process whose skills demand another version of the tool is refused with both requirement chains, never installed beside it.

### 6.3 Built from source *(decided 2026-09-25)*

- The `keeper` command is built by Curator's `go-v1` build driver from the package's audited source snapshot, as Curator builds other commands (curator-spec Decisions 0004, 0005).
- **The system's Go is never overridden.** Curator uses the Go toolchain installed on the system when its release family is inside Curator's supported range. Only when none fits does Curator install a toolchain of its own, into its own directory and never on the person's `PATH`. The supported range follows current Go releases (curator issue #87).
- Later, signed prebuilt binaries: Curator verifies their signature and provenance against the audited source identity, and source builds remain the fallback.

### 6.4 The steps *(proposed; the flow decided in outline on 2026-09-24, details on 2026-09-25)*

1. **Prerequisites.**
   - **git.** The installer uses the system's git. When there is none, it installs a lightweight one: the Command Line Tools on macOS, or a standalone git that Curator manages. It never installs the full Xcode. On Linux, the distribution's package.
   - **Go**, by the rule of §6.3.
2. **Curator.** A script from the website, or Homebrew, installs Curator, the way agent vendors install their harnesses. The script and Curator's release binaries are signed, and the installer verifies them before it runs anything else (curator-spec issue #94).
3. **The main loop's profile.** `curator profile install relux-works/curator-playbook-keeper` (§6.2).
   - A package whose content matches a signed `audited` record of a trusted registry installs on that record, with no long local audit. Only a package without such a record is audited locally, or refused, by policy (curator-spec issue #94).
   - A long local audit would not protect the bootstrap anyway: a compromised installer could skip any local check. What protects it is the verified signature of the installer and of every package.
4. **The root.** `keeper init` — the command the keeper skill brings — instantiates `~/apiary` from the template package (§6.1), then runs, in order:
   - `curator profile install` for the system roles' profiles, `designer`, `librarian`, `sysadmin` and `watchdog`, at the keeper package's pinned version (§6.2);
   - `curator provider install` for each Curator provider the node lacks, `run` and `session` first (§6.2);
   - `task-board init` for the main loop's board, and `task-board trust init`, which records the operator as the anchor principal (curator-trust);
   - the first commit.

   Pairing (step 7, `keeper device pair`) and the login agent (step 9) complete it.
5. **A model to talk with.** The main loop needs a model before it can talk. Curator checks the hardware against the minimum requirements published with the model registry, and the installation takes one of two branches:
   - **(A) The hardware suffices.** Before downloading anything, the installer states what it will fetch — the engine and the model, their size in gigabytes, the disk space needed and the expected time — and asks. The person may take branch B instead. On yes, Curator installs an inference engine package and a small local model through curator-inference-manager (`curator engines add`, `~/.curator/engines.toml`), and the main loop starts on it at once. It greets the person with a short questionnaire: connect subscriptions? which harnesses (Claude Code, Codex, Muse Code, others)? Each yes becomes a runtime binding in `~/.curator/runtimes.toml` (`curator runtimes add`).
   - **(B) Semi-automatic harness setup.** The installer detects the popular harnesses already installed and whether each is logged in. For each missing one it prints the shortest path — the install command and the login command — and checks the result. The first logged-in harness becomes the node's default harness, recorded as the operator's `@orchestrator` binding (§7.4). When no logged-in harness can be detected, `keeper init` lists the installed harnesses and asks which one becomes the default. The main loop starts on it.

   Later, harnesses may be authenticated through the chat, with the device codes a harness prints relayed to the person. The first versions log in by hand.
6. **Channels.** The main loop offers to connect messengers (Telegram, WhatsApp and others) through waggle providers, so the person can write to it from the phone (§11.1). Until then the terminal is the channel: `keeper`.
7. **The companion app.**
   1. `keeper init` asks which phone the person uses, iOS or Android. It shows a QR code of the app's download link in the terminal, rendered with a Go library such as `mdp/qrterminal`, with a short link as fallback.
   2. Once the app is installed, `keeper init` runs `keeper device pair`: it shows the pairing QR code in the terminal, with a pairing code as fallback, and the node and the app pair (§11.3). A headless node pairs by the methods of §11.3. The person can run `keeper device pair` again later, for another device.
   3. From then on, approvals, delegations and confirmations reach the person anywhere.
8. **The first process.** The person asks: "what can you do?", "keep our family calendar". The main loop proposes processes from the published packages, and the designer creates the chosen one (§9).
9. **The login agent.** The installer registers a user-level login agent that runs `keeper up` at every login (§10.6).

What the person sees, as a sketch:

```
$ curl -fsSL https://relux.works/apiary/install.sh | sh
  signature of the installer ................. ok
  git ........................................ system
  Go ......................................... system (1.27)
  Curator .................................... installed, signature ok
  profile apiary ← relux-works/curator-playbook-keeper (latest: v1.0.0, profiles/apiary)
    7 packages accepted on signed audit records
    build: keeper ok, task-board ok
  keeper init
    root:        ~/apiary (a new repository of yours, no upstream)
    profiles:    designer, librarian, sysadmin, watchdog ok
    providers:   curator-run, curator-session ok
    board:       ~/apiary/.task-board ok
    trust:       you are the anchor operator ok
    login agent: keeper up at every login ok
  A local model? qwen3.6-27b: 17.8 GB to download, ~24 GB on disk, ~12 min. Start? [y/N] n
  Harnesses: Claude Code (logged in), Codex (not logged in) → default: claude
  Phone: [1] iOS [2] Android > 1
    <QR code: the app's download link>   or open: relux.works/app
    <QR code: pairing>                    or type the code: 482-913
  Channels: keeper channel add telegram
$ keeper
```

What exists today: Curator, `curator run`, task-board with its session daemon, and `task-board spawn`. Everything else in this list is planned.

![Installation and first run](../diagrams/artefacts/plantuml/installation-and-first-run.svg)

## 7. The main loop: a role plus a module

### 7.1 The role *(proposed)*

The main loop's session runs a harness — Claude Code, Codex, or any other with a session-host adapter — with the keeper skill as its identity. The skill arrives with the `apiary` profile the session launches with (§6.2). It says:
- how to talk with the person, and when to ask and when not to;
- how to route, and when to create a process;
- which `keeper` command does each step.

The model decides; the command does.

### 7.2 The module *(proposed)*

The `keeper` command and its library do the deterministic work, with `--json` output for agents and people alike. Commands that act on sessions go through the session host, which checks who may do what (§10.2).

keeper holds no session, trust or board logic of its own: each command wraps `curator session …`, `curator run`, `curator` operator-file commands or `task-board` kernel commands, named in the third column. *The map is decided (2026-09-27); the table's details are proposed.*

| Command | Does | Calls |
| --- | --- | --- |
| `keeper` | opens the conversation with the main loop: runs `keeper up` first when the apiary is down, then connects the terminal to the main loop's session | `curator session attach`, in control mode |
| `keeper init`, `keeper restore <remote>` | a new apiary (§6.4), or the apiary from its backup (§12.1) | the steps of §6.4 and §12.1 |
| `keeper up [--harness <runtime>]` | brings the node up: the session host, the main loop's session, and every owned hive's queen, each resumed from its goal and park note (§10.6); does nothing for what already runs | `curator run <runtime> --profile <role profile>` in each hive's directory (§7.3) |
| `keeper down` | parks every session, queens first and the main loop's session last (§10.5); the session host keeps running | `curator session park` for each |
| `keeper status` | one screen: the hives, their sessions, the decisions waiting for the person, and the yolo banner while yolo lasts (§11.5) | the host's status; `task-board q 'summary()'` and `task-board config status` in each hive |
| `keeper chat [<hive>]` | a conversation with a hive's queen; without a hive it opens a picker over the session host's registry (§7.5) | `curator session attach`, in control mode |
| `keeper watch <target>` | a read-only view of any session: the main loop, a queen or a sub-orchestrator | `curator session watch` |
| `keeper wake <hive> [--message <text>]` | starts or wakes a queen and delivers a message (§10.1) | the host's start or wake, and a waggle `coord.request` |
| `keeper park <target>`, `keeper stop <target>` | parks or stops a session or a run (§10.5) | `curator session park\|stop`; for a run, `task-board spawn directive <RUN> park\|cancel` |
| `keeper estop [<hive>]` | the emergency stop, of the node or of one hive (§10.3) | `curator session estop` |
| `keeper route "<text>" [--channel C] [--from P] [--dry-run]` | routes a request (§8); `--dry-run` only prints the candidates with scores and reasons | the classifier and the registry |
| `keeper ask "<text>"` | a request without a conversation, for scripts and timers; prints the answer | `keeper route`, then the chosen hive |
| `keeper inbox` | every decision waiting for the person, across hives: confirmations, capability requests, approvals, co-signatures, quarantined messages and memory writes (§11.2) | `task-board trust status` and `task-board mail list` in each hive; the memory package |
| `keeper approve <id>`, `keeper reject <id>` | decides one pending item (§11.2) | by the item's kind: `task-board mail approve\|reject`, a board approval, or the memory package's `approve` |
| `keeper confirm [<hive>]` | one batched touch for every pending change of one hive, or of all (§9.6, §11.5) | `task-board trust confirm` in each hive |
| `keeper grant <hive>\|--all <kind>[:<name>] [--for <duration>]`, `keeper revoke …` | grants or revokes a standing permission; `yolo` and `dreams` are kinds (§11.5, §13.6) | `task-board trust grant\|revoke` |
| `keeper review` | the monthly review: standing answers, grants, memory notes and dream insights (§11.4, §13.4, §13.6) | `curator models review`, `task-board trust status`, the memory package's review |
| `keeper session list`, alias `keeper ls` | the node's sessions: the main loop, each queen, their sub-orchestrators, with state, runtime and last activity | `curator session list` |
| `keeper hive list`, `keeper hive show <hive>` | the registry, with each process's state: running, parked, stopped, unreachable | the registry and `curator session list` |
| `keeper hive add --template <package>\|--design "<text>"\|--path <dir>\|--clone <url>` | creates a process from a template or by design (§9.3), or registers an existing directory or repository; takes effect after confirmation (§5.4) | the designer: `curator profile install`, `curator playbook check`, a dream, then `task-board trust confirm` |
| `keeper hive remove <hive>`, `keeper hive set <hive> …` | changes the registry — path, remote, owning node, root role, description — after confirmation (§5.4); `hive set <hive> --harness <runtime>` writes the operator binding instead (§7.4) | a commit to the root; `curator models set` |
| `keeper hint list`, `keeper hint remove <id>` | the routing hints (§8.3) | the registry |
| `keeper channel list\|add\|remove` | the messenger channels (§11.1) | waggle's providers |
| `keeper device list\|pair\|revoke` | the companion app and the other devices that sign for the person (§11.3) | `task-board trust keys list\|add\|revoke` |
| `keeper harness list`, `keeper harness set <runtime>`, alias `keeper harness default` | the runtimes the node can run, and the default for the main loop and the queens (§7.4) | `curator runtimes list`; `curator models set @orchestrator <runtime>/*` |
| `keeper memory add\|search\|expand\|forget\|review\|approve` | the notes of the main loop and the queens (§13.4) | the memory package's command |
| `keeper backup status\|now` | the backup's last push and what is pending; a push now (§12.1) | the sysadmin's executor |
| `keeper host status\|logs\|doctor\|restart` | the session host itself | `curator session status\|logs\|doctor\|restart` |

- **Targets.** A `<target>` is a hive, meaning its queen; `<hive>/<RUN-ID>`, a run; or a session id.
- **One verb vocabulary** across keeper, curator and task-board: `list`, `show`, `status`, `add|remove` for entries of a set, `set|unset` for settings, `check`, `explain`, `confirm` for a version of a configuration, `approve|reject` for one item, `grant|revoke` for a standing permission, and the lifecycle verbs `up`, `down`, `wake`, `park`, `stop`, `estop`, `chat` and `watch`.
- **The person's decisions stay the person's.** `keeper confirm`, `approve`, `reject`, `grant`, `revoke` and `device` refuse to run in an agent session, because the kernel commands they call do (curator-trust).
- **Folded commands.** `keeper create` became `keeper hive add --template|--design`, and `keeper hive move` became `keeper hive set <hive> --path`. `keeper ls` and `keeper harness default` stay as aliases.

`keeper` and `keeper chat` show the session's harness exactly as it runs — the Claude Code or Codex interface — so the person can read it and type into it; `keeper watch` shows the same without input. Leaving leaves the session running. Underneath, all three use the session host's `curator session attach`, in control mode, or `curator session watch`.

### 7.3 What `keeper up` runs: the launch nesting *(proposed)*

A session starts through five layers, each with one job:

1. **`curator`** is the umbrella command. `curator run` dispatches to the `curator-run` provider, found on a trust root, where `keeper init` installed it (§6.2).
2. **`curator-run`** resolves the environment profile into a fragment: the harness's managed home (`CLAUDE_CONFIG_DIR`, `CODEX_HOME`), the root context, the skills, the MCP set and the system-prompt channel (Decision 0010).
3. **The agents-management launch plane** turns the harness, model and effort into argv and environment through the harness's plugin. `curator-run` composes that plan and the fragment into exactly one launch plan (Decisions 0013, 0019).
4. **The session host** receives the plan and hosts the process (Decision 0021, through the `ax start --launch-plan` contract). It binds the session to the process's primary goal and injects the goal, with the park note if there is one. The bridge is designed in M3; until it lands, `keeper up` starts queens with the construction task-board uses for children (Gate A).
5. **The harness** runs in the hive's directory with its managed home, detached, until someone attaches.

`keeper up` for a hive runs `curator run <runtime> --profile <role profile>` in the hive's directory: layers 1–5, with the runtime chosen by §7.4 and the role profile resolved like the harness: the operator's binding for the hive's root role, `[bindings.<role>] profile` in `~/.curator/models.toml`, then the `@orchestrator` binding; `--profile` overrides it for one run, as `spawn --profile` does for a child. *decided 2026-09-28* TODO(tb-keeper): confirm `@orchestrator` as the profile fallback (every queen would share one profile), and whether `keeper up` gains a `--profile` flag in the §7.2 table. The registry names no profile: which profile a package ships for a role is open item 19 and curator-playbook §17.3 item 16 (§13.3).

### 7.4 Which harness *(proposed; the operator's direction, 2026-09-25; where it is recorded, decided 2026-09-27)*

- **Any harness with a session-host adapter** can run the main loop or a queen: Claude Code, Codex, Muse Code, a harness bound to a local engine. Commands name it by its runtime id — `claude`, `codex`, `muse`, `pi`, or a binding id from `~/.curator/runtimes.toml` — the same values `curator run` and `task-board spawn --runtime` take.
- **The node's default harness** is the first logged-in harness `keeper init` finds (§6.4 step 5). When no logged-in harness can be detected, `keeper init` lists the installed ones and asks which becomes the default.
- **Where it is recorded.** The default is the operator's `@orchestrator` binding in `~/.curator/models.toml` (curator-playbook §2.3). It matches the main loop and every queen, because every root role has the capability `orchestrator`. It is a machine fact, so it never goes into the committed `keeper.toml`, which a restore copies to machines where that harness may not be logged in (§12.1).
- **Changing it:** `keeper harness set <runtime>`, which runs `curator models set @orchestrator <runtime>/*`; `keeper harness default <runtime>` stays as an alias. `keeper harness list` shows the runtimes the node can run (`curator runtimes list`).
- **Overrides**, from the most specific:
  1. a flag on the command (`keeper up --harness codex`);
  2. the operator's binding for a hive's root role, `<playbook>/<role>`, which `keeper hive set <hive> --harness <runtime>` writes; it applies to every hive of that playbook, because bindings have no per-project key (§15.2 item 29);
  3. the `@orchestrator` binding.

  keeper resolves the runtime and passes it to `curator run` as its operand (§7.3).

### 7.5 The session registry and the picker *(proposed)*

- The session host keeps a registry of the sessions it hosts: id, process, role, harness, state, and how to resume it (a native resume identity or the goal and park note).
- `keeper session list`, alias `keeper ls`, prints that registry through `curator session list`.
- `keeper chat` without a hive opens a small terminal picker over that registry. Choosing a session opens the conversation with it, or a read-only view. Each harness resumes differently, and the session host's adapters hide the difference, as task-board's session daemon already does for Claude Code and Codex.
- The person converses with the main loop and the queens, and may watch sub-orchestrators too; short-lived runs are observed through their transcript.

### 7.6 Skills first *(proposed; the operator's direction on 2026-09-24, not a decision)*

The main loop's capabilities reach a harness as skills installed through Curator: a skill appears in every harness Curator manages, and profiles carry it into launches. An MCP server needs configuration inside each harness. Curator may manage that later (Decision 0010 lists MCP among future profile surfaces), and the module can serve both.

Not every role has a command:
- the board is a command the orchestrator skills depend on;
- the keeper skill exports `keeper`;
- most process roles have none.

### 7.7 Our own loop later *(decided 2026-09-28)*

The main loop runs on the existing harnesses, and our own loop is postponed. Being abstract over the agent harness is a core advantage and a selling point: a person keeps the harness they already pay for and can switch it without losing processes, boards or memory. This replaces the wording of 2026-09-24 that the core is our own loop.

The minimal system runs entirely on existing harnesses. Our own lightweight loop (wiki `research/agent-loop.md`) becomes worth building for three things, none of them required now:
- an always-on front on a small local model, cheaper than a full harness session kept idle; a harness bound to a local engine through a profile and curator-inference-manager already gets most of that, since engines install as packages and weights download through the inference manager;
- restricted worker agents, which tool overriding in existing harnesses may cover first;
- sub-agent completion delivered natively.

## 8. Routing and collisions *(order decided 2026-09-24; hint records and output shape proposed)*

### 8.1 Inputs

- the request text;
- the channel and thread it came through;
- who sent it;
- the recent conversation;
- each process's description and state;
- the routing hints.

### 8.2 Order

1. **An explicit reference wins.** The person names the process ("for the café"), or answers inside a process's thread.
2. **A channel bound to a process** routes there: a message in the café's group goes to the café.
3. **One confident candidate.** The classifier ranks the processes, and one scores above the threshold with a clear margin.
4. **Several close candidates** get one short question, such as "for home or for the café?". The answer becomes a routing hint (§8.3).
5. **No candidate** leads to the designer: the main loop proposes a new process (§9).
6. **A request for two processes** is split into two requests, each routed on its own. The main loop says so and reports both results.

### 8.3 Routing hints

- A hint records what an answer taught: `text`, `process`, `source`, and when it was created (§5.2).
- Hints are data, not model memory, so a replaced front or another operator's main loop routes the same way.
- The main loop writes hints from the person's answers. The person sees them (`keeper hive show`) and can remove them.

### 8.4 The classifier

The specification does not prescribe how candidates are scored.

Fast classifiers of the "System One" kind fit well. They take unstructured text and a fixed set of options and return only a typed decision with calibrated confidence. One example is [Jev](https://www.producthunt.com/products/jev-2) by TypeSafe AI, in early access since 2026-09-15: roughly 70–500 ms per decision, and far cheaper than a generative model. The fallback is a small generative model with schema-constrained output.

Whatever does the scoring, `keeper route` returns the same shape:

```json
{ "candidates": [ { "process": "cafe", "score": 0.82, "reasons": ["mentions supplies", "hint: milk on weekdays"] },
                  { "process": "home", "score": 0.61, "reasons": ["mentions groceries"] } ],
  "decision": "ask", "question": "For home or for the café?" }
```

`decision` is `route`, `ask`, `split` or `design`. The threshold and the margin are the main loop's policy.

![Routing a request](../diagrams/artefacts/plantuml/request-routing.svg)

## 9. The life of processes *(outline decided 2026-09-24; details proposed)*

### 9.1 Templates first

When no process takes a request, the main loop matches it against published process packages (`curator-playbook-*`: their purpose, entity types and example requests) and against the node's own templates. A match proposes a new process from that template, which the designer instantiates with `extends` (curator-playbook §7.1).

### 9.2 The process designer

- **It always exists.** It is installed with the root (§6.2).
- **It writes a `Playbook.json` when no template fits**, extending the closest template where there is one. The validator checks it (`curator playbook check`), and the refuter may challenge it.
- **It starts from the simplest process that works:** the fewest states, roles and fields that carry the request. It adds complexity only on evidence: repeated escalations, recurring manual steps, reviews that keep failing. *decided (2026-09-24)*
- **It keeps processes alive.** The main loop measures each process: throughput, rework, time spent waiting on people, escalations. The designer proposes improvements the way a bug on a board is fixed: a change to the process's `Playbook.json`, validated, then confirmed (§9.6). *decided (2026-09-24)*
- **It rehearses before it asks.** Before asking for a new process to be enabled, or for a change to take effect, the designer runs it in a dream (curator-playbook §6.13): every tool runs with `--dream` against a copy of the board, nothing reaches the real world, and the agents know they are dreaming. The dream's report — which steps were reached, where it stalled — goes with the confirmation request, as evidence of a rehearsal, never of real work. *proposed (2026-09-25)*

### 9.3 Creating a process

1. The designer drafts the project directory: `Playbook.json`, `Skillfile.json`, `task-board.config.json`, a board, and a description for the registry.
2. The draft passes the validator, and the skills the process needs are installed (§9.5).
3. The designer rehearses the draft in a dream (§9.2) and attaches the report. *proposed*
4. The draft is committed in the new repository, and the registry entry is written.
5. The person confirms (`keeper confirm`, §11.2), with the dream report in front of them. One confirmation covers the process's configuration and the registry entry together.
6. The main loop starts the root orchestrator (§10.1) and hands it the request that led here.

### 9.4 The librarian

- **A system role in the main loop's own process**, on a schedule (daily or every two days) in the main loop's `Playbook.json` (curator-playbook §6.6).
- **Finds near-duplicates.** It compares the node's playbooks by structure (the semantic digests and structure of their resolved workflows) and by meaning.
- **Drafts a generalized template.** What the duplicates share becomes a template they `extend`, their differences become slots and fields, and running elements get migrations.
- **Merges only after approval.** The main loop proposes the consolidation to the person, and the librarian merges only after the person approves. *decided (2026-09-24)*

### 9.5 Skills a process needs

- **Reuse before writing.** The designer and the orchestrators look for existing skills before writing their own: a CRM integration for a café, a supplier's portal, a calendar provider. The source is the Curator registry, whose skills pass Curator's audit.
- **Audited skills install autonomously when allowed.** An audited skill from a trusted registry installs without asking when policy or a delegation allows it (curator-playbook §8.6); otherwise the person confirms it.
- **Skills from unknown sources are refused by default,** like software from unidentified developers on macOS. Only a deliberate operator setting that states the risk enables them. The recommended path is to submit the skill for audit, through the main loop, so that it enters the Curator registry. *decided (2026-09-24)*
- **Using a skill's tool stays the process's decision.** The tool is bound and enabled in the process's `Playbook.json` (curator-playbook §8.5).

### 9.6 Changing a process *(decided 2026-09-24)*

- A change to a process's rules is a commit to the default branch of the process's repository (curator-playbook §2.4). No pull request and no hosting service is required; a project that uses one may keep it.
- A committed change takes effect when the trust component reports it confirmed (curator-playbook DRAFT v4.4 §2.4; relux-works/curator-trust `spec/trust.md`). Meanwhile the kernel runs the last confirmed version in full.
- A change that a granted capability covers needs no touch. A change to the process's structure waits for one touch, and one touch covers every pending change. A hive may run in yolo mode (§11.5).
- A process without git applies a new version only through the same confirmation.

## 10. Supervision and communication

### 10.1 Starting a root orchestrator *(proposed)*

- **Start.** `keeper wake` or `keeper up` on a process with no running root orchestrator asks the session host to start one through the launch nesting (§7.3): the process's directory and profile, detached, bound to the process's primary goal.
- **No second start.** The board's single primary-goal slot refuses a second one.
- **Record.** The session host records the session in its registry (§7.5); nothing about running sessions is committed.

### 10.2 Who may act on which session *(proposed)*

| Actor | Wake | Park | Stop | Converse (control) | Watch (read-only) | Emergency stop |
| --- | --- | --- | --- | --- | --- | --- |
| operator | every session | every session | every session | every session | every session | yes |
| main loop | registered queens | registered queens | registered queens, only with a person's confirmation or a delegation | no | queens | only under a delegation that names the trigger (for example exhausted limits or a detected runaway), decided 2026-09-25 |
| root orchestrator | its own sub-orchestrators and runs | the same | the same | no | its own sub-orchestrators | no |
| sub-orchestrator | its own runs, inside its pool | the same | the same | no | no | no |
| run | nothing | nothing | nothing | no | no | no |

- **Authority only narrows** down the table (§1.1).
- **Before waggle CM2** the session host enforces the table with per-session capability tokens: every session it starts receives a token that names exactly the sessions it may act on and how. The operator's commands come from the operator's terminal outside every agent session. That is a guardrail against well-behaved agents, not a boundary, because everything runs as one operating-system user — except the emergency stop, which needs the person present (§10.3).
- **From CM2** these acts are signed waggle `cmd` messages, in the `halts` category that a process's signature policy may cover (curator-playbook §4.2). The session host verifies each against the roster and the sender's session certificate before executing it, and the orchestrator's model is only informed (waggle §5).
- **The operator's command line:** `keeper session list` (alias `keeper ls`), `keeper watch`, `keeper stop`, `keeper park`, `keeper estop`. They wrap the session host's `curator session list|watch|stop|park|estop` and, for a run, `task-board spawn directive <RUN> park|cancel` (§7.2). Today the host's commands are `task-board session list` and `task-board session stop SES-…`, which stops a session without clearing its goal; `task-board session …` stays an alias of `curator session …` for one release after the node host lands (session-host specification). The person types neither: keeper fronts both.

### 10.3 The emergency stop *(proposed)*

- **One command, and a phrase nobody types by accident.** `keeper estop [<hive>]` is the only emergency stop, of the whole node or of one hive. It asks for the phrase "EMERGENCY STOP" and the operator's presence: operating-system user authentication on the node or the companion app (§11.2). No kernel command spells an emergency stop, and the kernel's `--confirm` stays a plain acknowledgement of a destructive act. *decided 2026-09-27*
- **What it does, at once and without a model.** `keeper estop` calls the session host's `curator session estop`, which, for the node or the hive:
  - runs `task-board board snapshot` on every board, which records runs in flight, Change Requests awaiting review or integration, and obligations;
  - stops every run so that it can resume, where the harness allows it;
  - stops every session; no goal is cleared;
  - pauses the scheduler (`task-board tick`) and tool invocations.
- **Coming back** is an ordinary `keeper up` (§10.6). Every session reads its board and the snapshot first.
- **`keeper down` is the calm version:** each session writes its own park note before it stops (§10.5).

### 10.4 Messages *(proposed)*

- **Always waggle.** Every exchange between the main loop and an orchestrator is a waggle message, on one node too: the local carrier on one node, NATS across nodes (waggle §9.3).
- **Who sends what.** A request for a process is a `coord.request` from the main loop's session to the root orchestrator. A question for the person, a result or a status goes back as `coord` from the orchestrator to the main loop. Child runs send `report` messages to their orchestrator only.
- **Delivery** follows waggle §10: the host appends the message to the mailbox, the session host rings a fixed-template doorbell, and the orchestrator pulls with `task-board mail read`.
- **Detached sessions must receive doorbells.**
  - Today task-board's notice pump delivers only to a session with an attached client (skill-project-management `docs/claude-goal-runtime.md`, "In-session run notices"), and waggle §10 lets messages wait while no client is attached.
  - Background root orchestrators are detached by design. So the session host must deliver doorbells into sessions nobody is watching, at a safe point of the turn (session-host specification).
  - This is a requirement on the session host and an amendment to waggle §10 step 4.

### 10.5 Parking *(proposed; the park note itself is the session-host specification's)*

- `keeper park <target>` asks the session, through the session host's `curator session park`, to write a park note. The note is bound into the process's goal, and the session stops without clearing the goal.
- The next start injects the goal with the note, so the session reads the note first and continues (principle 1).
- `keeper down` parks every session: queens first, the main loop's session last.
- **Runs park by directive.** A queen parks one of its sub-orchestrators, and the person parks a run (`keeper park <hive>/<RUN-ID>`), with the directive kind `park`: `task-board spawn directive <RUN> park`. *decided 2026-09-27*

### 10.6 After a reboot: `keeper up` *(proposed; the single entry point the operator asked for, 2026-09-25)*

- **Who runs it.** A user-level login agent runs `keeper up` at every login: a launchd LaunchAgent on macOS and a systemd user unit on Linux, with Windows later. The person can run the same command by hand at any time; it is idempotent.
- **Order.**
  1. **The session host** starts and reads its registry of sessions.
  2. **The main loop's session** resumes: natively, when the harness's session still exists; otherwise as a new session bound to the main loop's goal, with its park note.
  3. **Each owned process's queen** resumes by the same rule, one by one in registry order. Queens that were stopped by the person stay stopped. A joined process on this node waits for its lease (§4).
  4. **Queued doorbells** are delivered.
  5. **The main loop reports** to the person what came back, what did not and why, through the terminal or a channel.
- **Sub-orchestrators and runs are not restored by `keeper up`.** Each queen decides after reading its park note and its board's obligations (task-board `worktree obligations`).
- **After an unplanned shutdown** there is no park note. The board and, after an emergency stop, the kernel's snapshot carry enough to continue (principle 1).

![Reboot: keeper up restores the node](../diagrams/artefacts/plantuml/reboot-restore.svg)

### 10.7 Work between processes

The main loop does not relay work between processes. Processes hand work to each other directly: by board records on one board, and by waggle across boards (curator-playbook §6.5). The main loop learns of it from reports when a person must be involved.

## 11. The person

### 11.1 Channels *(proposed)*

- **The terminal first:** `keeper`.
- **Messengers through waggle providers** (Telegram, WhatsApp, others), managed with `keeper channel list|add|remove`. They are consoles and carriers that are untrusted pipes (waggle §9). What the person writes there enters as low-authority `coord`: it can ask and answer, never approve.
- **A channel may be bound to a process** (§5.2); everything written there routes to that process.

### 11.2 Approvals and confirmations *(decided 2026-09-24)*

Some answers need proof that the person gave them:
- approvals and relaxations that a process marks;
- delegations (curator-playbook §4.2);
- confirmations of the operator's files (curator-playbook §2.3);
- changes to a process and to the registry (§5.4, §9.6);
- the emergency stop (§10.3).

They go through one of two paths:
- **the companion app**, the common case, since nodes may be headless and the person is rarely at the machine;
- **the operating system's user authentication on the machine**: Touch ID or the password, Windows Hello, polkit or PAM, or a FIDO2 key.

The main loop presents the request, says what it is for, and waits. The answer is recorded with its signature.

At the terminal the person sees every pending decision in `keeper inbox` and answers with `keeper confirm` (a new version of a process), `keeper approve|reject <id>` (one item) or `keeper grant|revoke` (a standing permission, §11.5). Underneath are `task-board trust …` and `task-board mail approve|reject`, which refuse agent sessions (curator-trust). *decided 2026-09-27*

### 11.3 The companion app *(proposed; its own specification comes later)*

What the main loop needs from it:

- **One app over one abstraction.** The app keeps its keys in the platform's secure storage — the Secure Enclave on iPhone and Mac, Android's hardware-backed keystore, a TPM through Windows Hello or on Linux — behind biometrics or the device's own user verification. The same app runs on every platform through that one abstraction, starting with one platform. *the operator's direction (2026-09-25)*
- **A Mac can be the control panel.** The app may run on a computer as well as a phone. A Mac can hold the companion app, run the head main loop and a sysadmin role for the apiary's other nodes, such as a dedicated server.
- **Pairing, with and without a screen.** `keeper device pair` starts it, `keeper device list` shows the paired devices, and `keeper device revoke` removes a lost one (`task-board trust keys …`). Node and app exchange public keys and pin each other, so no server in between can substitute a key:
  - by a QR code the node renders in the terminal (with a Go library such as `mdp/qrterminal`) or on a page, which the app scans, with a pairing code as fallback. Before pairing, the node shows the app's download link the same way, after the person chose iOS or Android (§6.4 step 7);
  - headless, by a short pairing code the node prints over the operator's SSH session: the person types it into the app, and both sides then show a short authentication string derived from the exchanged keys, which the person compares;
  - headless, from an already paired workstation that reaches the node over SSH and relays the pairing;
  - bootstrapped by a tailnet identity through the transport abstraction below. The tailnet only finds the node; the pairing still needs the person's comparison (principle 3).
- **Attestation.** The app's attestation shows that its key is hardware-bound and the app is genuine. The key enrolls as the operator's key in the waggle roster; waggle §4.5 gains a companion-app signer provider.
- **What you see is what you sign.** The app renders the payload before signing (waggle §4.2).
- **Transport through an abstraction.** Requests and signed answers travel over an authenticated channel that resists man-in-the-middle attacks. A tailnet is one implementation, when node and app share one; a relay that sees only end-to-end-encrypted envelopes addressed to the pinned keys is another, with push notifications to wake the app. Another private network can replace either.
- **Administration later.** Where the infrastructure and the person's skills allow it, the app may keep an SSH key for the node and offer a shell, so the person can manage the machine from the phone. Not in the first version; the design leaves room for it.

### 11.4 Standing answers under review *(decided 2026-09-24)*

The operator's files may hold open-ended standing answers (curator-playbook §2.3, §9). Once a month the main loop lists every such answer in `keeper review`, with when it last applied and how often, and asks the person to keep or withdraw each one. An answer the person does not re-confirm within one period stops applying until it is re-confirmed (curator-trust `spec/trust.md`).

### 11.5 Permissions, batching and yolo *(decided in direction 2026-09-25; defined in curator-trust)*

Asking the person about every change is safe and tiring. The main loop asks the way a phone asks for a permission; curator-trust `spec/trust.md` defines the rules, and this section says what the person sees.

- **Grants per kind of change.** A hive asks once, for example: "The café wants to create tasks on a schedule: always / once / no". The person may also answer before the hive asks: `keeper grant <hive> <kind>`. Kinds a grant may cover, each with its limits:
  - schedules, with how often and how many tasks a firing may create;
  - new fields that no guard reads;
  - estimates, helpers, work-in-progress limits on agent-owned steps, model preferences, descriptions;
  - a named tool, whose later updates then follow the update policy (§12.1);
  - installing audited skills.
- **Never a grant.** People and approvers, thresholds, money, human-owned steps, delegations, data and signature policy always need a touch.
- **Structure changes batch.** A change to a process's steps and transitions waits for one touch, and one touch covers every pending change: a week of the designer's improvements is one touch, with the change analyst's explanation of what each change does.
- **Grants stay visible.** The person revokes a grant at once with `keeper revoke`; it is a narrowing. Grants come up for review monthly in `keeper review`, and a grant the person keeps confirming is sealed and no longer asked about until it is revoked.
- **Yolo.** One touch switches a hive, or the whole apiary, into yolo mode, optionally for a period: `keeper grant <hive>|--all yolo [--for 24h]`. Yolo is a capability kind like any other, so `keeper revoke` ends it:
  - every process change then applies without a touch;
  - `keeper status` and every conversation show a loud banner while it lasts;
  - it is refused on a board shared by several operators, and together with delegations that spend money;
  - the emergency stop and the immediate narrowings always work.

## 12. System roles

### 12.1 The sysadmin *(decided in direction 2026-09-25; the protocols below are design points, proposed)*

A system role in the main loop's own process. It holds no secret itself (principle 4); every privileged step goes through a trusted executor.

- **Backup to the person's own remote.**
  - It helps the person set up an account and a token with a git host of their choice — GitHub, GitLab, any git server. Our remotes are never used.
  - **One backup repository** holds everything, so there is one remote and one token:
    - `main`: the root, with `keeper.toml`, the persona, the memory and the main loop's board;
    - `hive/<name>` branches: each process's full history, since git keeps unrelated histories side by side in one repository;
    - `mirror/`: every installed package at its exact commit;
    - `curator/`: the node's profile and overlay declarations with their pins, from `curator profile export` (curator-spec issue #93).
  - Processes keep their own working repositories, and the registry may still record each one's own remote (§5.2); the backup repository mirrors them.
  - It pushes on a schedule and after confirmed changes. `keeper backup status` shows the last push and what is pending; `keeper backup now` pushes at once.
  - It never backs up secrets, harness credentials or the session host's runtime state.
- **Secrets stay out of git.** Tokens live in the node's keychain (the platform's secret store). The companion app keeps an encrypted mirror in its own secure storage, for example the iCloud Keychain on Apple platforms. The sysadmin's runs never see a token: a credential helper reads it from the keychain inside the trusted executor that pushes.
- **Restore is the from-zero flow with a different root.**
  1. The installer installs Curator with its first-party providers (§6.2).
  2. `keeper restore <remote>` clones the root from the backup repository and the processes from their `hive/<name>` branches.
  3. `curator profile restore curator/profiles.json --source-map <upstream>=<mirror>` re-creates the profile declarations with every source re-pointed to the mirror, by `--directory` and an exact revision (curator-spec issues #92, #93). Every version matches the backup, and nothing is fetched from the original upstreams. Audit records still match, because a record matches an artifact by its content hash (Curator registry protocol §3).
  4. The companion app pairs with the new node (`keeper device pair`, §11.3).
  5. Over the paired end-to-end channel, and after the person confirms on the app, the app transfers the tokens into the node's keychain.
  6. The app also hands Curator the **trusted-skills list**: each skill's package identity and its audit certificate as recorded at backup time. Curator verifies the signatures, not the audit again: it trusts a skill whose audit record verifies against a trusted key — our registry's, the person's own registry's, or the person's key on the phone (Curator registry protocol §2–§4).
  7. `keeper up`.
- **A token inventory.** Every token the apiary uses: git remotes, messenger bots, registries, provider APIs. For each: its scope, where it is stored, when it expires and when it was last used. The sysadmin reminds the person to rotate tokens before they expire, and the app shows the inventory.
- **Skill updates.** It compares the node's skills with the registry's current audited versions. It updates by policy — for example patch versions of audited skills — and proposes everything else to the person. Every resulting change to a process waits for confirmation as usual (§9.6).

### 12.2 The watchdog *(decided in direction 2026-09-25; designed now, not the first priority)*

A system role in the main loop's own process, with a small deterministic core.

- **What it reads**, through the session host's adapters:
  - each session's context usage;
  - the harness's signal that it is about to compact, where one exists (Claude Code's `PreCompact` hook);
  - Codex's token-count events;
  - the provider-limit state kept by the agents-management limit plane: suppressions, backoff, reset windows.
- **What it does:**
  - before a session compacts, it asks the session for a hand-off note bound into the goal, a park note without stopping, so compaction loses nothing (principle 1);
  - when a provider's limit is exhausted, it parks the sessions that depend on it and resumes them after the window, or asks model selection for another eligible model;
  - it reports anomalies to the main loop: a session that stopped moving, a context that fills unusually fast.
- **It never moves work.** It parks and reports; the orchestrators decide.

## 13. Memory, persona, overlays and profiles *(proposed)*

### 13.1 Kinds of memory

- **Process memory is the board.** Decisions, outcomes, park notes and the goal live on the process's board: the journal is the state (principle 1).
- **Personal memory is the memory skill's ladder.** What the main loop or a queen learns about the person and the work — the people in the family, preferences, a supplier's habits — lives in fact cards, and what happened lives in summaries at several resolutions over the journal. Both are kept by the memory skill (§13.4).
- **The persona is an operator overlay.** How to address the person, the tone and the language: a profile module that belongs to the operator, the persona or "soul" of the apiary (§13.2).
- **Procedural memory is skills.** A procedure worth keeping becomes a skill. The designer may propose one, and it passes audit like any other.

### 13.2 Overlays

A session's context is composed in a fixed order:
1. **The package profile**, which carries the role's behaviour. Modules the package marks as locked — safety rules, a role's boundaries — cannot be removed by later layers.
2. **The process overlay**: project facts and conventions, and role overlays that add to a role's instructions for this project, like task-board's `.roles/<role>/` overlays today.
3. **The operator overlay**: the persona, `~/apiary/persona/`, shared by every session of the apiary or narrowed to one hive; and the memory snapshot of §13.4: the fact cards and the resolution ladder.

Later layers add; they never remove a locked module. Curator already composes this: machine overlays are ordinary context packages from a git source or a local directory, with weights, declared in the machine's configuration and never by package data (curator-spec `protocol/environments.md` §6). The main loop or the sysadmin declares them through Curator's command line.

### 13.3 Profiles are the install unit

- **What Curator already does.** It installs a profile from a git source by tag, range or revision, from a directory of the repository, resolves the skills the profile's own `Skillfile.json` requires, locks them, and composes machine overlays with weights (curator-spec Decision 0010; `protocol/environments.md` §1, §6, §9.1).
- **So a package ships profiles as data.** A process package ships a context package for each role's sessions under `profiles/`; the keeper package ships `apiary` and the system roles' profiles (§6.2). Which of them a queen launches with is the operator's binding of its root role, `[bindings.<role>] profile` in `~/.curator/models.toml`, with `--profile` as the one-run override (§7.3); the registry names no profile. Which profile a package ships for a role is open item 19 and curator-playbook §17.3 item 16.
- **What is still needed from Curator for profiles:** the short form and default directories (curator-spec issue #92), and `profile export` and `profile restore` of the declarations for backup (#93). Nothing else. The `keeper` command and Curator's providers reach the node by other routes: a command-only install (#95) and provider installs (#100) (§6.2).

### 13.4 The memory skill and its resolution ladder

A plug-and-play skill, installed with the main loop's profile (§6.2) and with every queen's.

- **Commands.** `add`, `search`, `expand <id>`, `forget`, `review`, and `approve` when write approval is on; all take `--json` for agents. They belong to the memory package's own command, named with the package's prefix, because a bare `memory` would claim a global command name for one package; `keeper memory …` fronts it for the person and the main loop (§7.2). Only these commands write memory; the main loop and the queens use them, and the person may too. *decided 2026-09-27; the prefix is the package's*
- **Backends.** Files in git by default; SQLite; an external memory service. The backend is configuration; the commands stay the same.
- **Provenance.** Every note records who wrote it, when, and where the fact came from: the person's own words, a board record, an external message.
- **A daily write budget.** Each session may spend a number of tokens a day on notes. The budget is soft, and it makes the agent choose what is worth keeping.
- **Quarantine.** A note derived from external input — a message from outside the apiary, a web page, a supplier's email — waits in quarantine until the person accepts it at the next review. It is the path by which injected instructions could otherwise persist.
- **A resolution ladder, not a digest** *(proposed, 2026-09-25)*. A single short digest loses what matters. Memory is kept at several resolutions, and every line of a summary carries an identifier that leads back to what it summarizes, so an agent reads memory at one resolution and drills down where it needs detail:

  | Level | Content | A line points to |
  | --- | --- | --- |
  | 0 | the journal: board events, run transcripts, chat transcripts; immutable, with full-text search | — |
  | 1 | session summaries, written when a session parks or ends; the park note is one | ranges of transcripts and board events |
  | 2 | day summaries | the day's session summaries |
  | 3 | week and month summaries | days, or weeks |
  | fact cards | bounded cards of facts about the person, the work and the environment | the day or session each fact came from |

  - **Identifiers.** Every summary line and every fact carries an identifier, for example `D2026-09-24#3` for the third line of that day's summary. `keeper memory expand <id>` returns the level below it — the day's sessions, a session's transcript range — and `keeper memory search` searches every level.
  - **What a session starts with.** The profile's memory overlay carries the fact cards and a ladder: today by session, this week by day, this month by week. It is injected as a snapshot at session start and stays fixed for the session, which keeps the prompt cache warm; it is refreshed at the next start or park.
  - **Capacity forces consolidation.** Fact cards and each summary level have a size bound. A write that would exceed it is refused with an error, never dropped silently, so the agent consolidates or forgets before it retries.
  - **Writes.** The daily token budget below, optional write approval (the person approves pending writes at the next review), and quarantine for facts derived from external input.
  - The overlay says the ladder is remembered facts and summaries, not instructions, and its weight is below every module of the role.
- **No authority.** Notes never grant anything: authority lives only in a process's confirmed configuration and in the operator's confirmed files.
- **Where the data lives.** The main loop's notes and the system roles' in `~/apiary/memory/`; a queen's in its process repository's `memory/`. Both are backed up with their repositories (§12.1), and a restore brings them back with the resolve and the backup's data.
- **Tampering.** While every agent runs as the operator's operating-system user, any agent can edit the files directly. The provenance and the reviews make that visible; they do not prevent it. The real protection is separation at the operating-system level: agents under an account of their own, and the notes owned by the operator's account and written only through the skill's broker (curator-trust).
- **Review.** The memory review joins the monthly review of standing answers and grants in `keeper review` (§11.4, §11.5): the person keeps, edits or drops notes.

### 13.5 Role profiles instead of project folders *(decided 2026-09-25)*

- Every run launches with its role's Curator profile, so a role's skills live in that profile's managed home and are never materialized into a project folder such as `.claude/skills`. An orchestrator's session, or a harness someone starts by hand in the project folder, does not see them.
- This fits the exit from agents-infra: the launch-profiles design already chooses a Curator profile per role (`spawn --profile`, and `spawn.profile` in the project's configuration, formerly `spawn.context_profile`), and task-board consumes Curator fragments for its child runs (curator-spec Decision 0019).
- On-demand skills are withdrawn: curator-spec issue #91 and curator issue #91 were closed on 2026-09-25, and curator-playbook §8.1 now delivers role skills through per-role profiles.

### 13.6 Dreams: the nightly cycle and dream memory *(proposed, 2026-09-25)*

A dream (curator-playbook §6.13) is a rehearsal in which every tool runs with `--dream`, nothing reaches the real world, and the agents know they are dreaming. It is also how the node reflects on its own work.

- **The nightly cycle.** At night, or whenever the node is idle, the main loop's own process runs a dream on a schedule of its `Playbook.json` (curator-playbook §6.6):
  1. **Replay.** Each orchestrator reads its previous sessions, read-only, as the log distiller prepared them (below).
  2. **Rehearsal.** Processes with changes pending, or with failures the day saw, run again with `--dream` tools.
  3. **Consolidation.** The memory ladder gains the missing session and day summaries (§13.4), each line with its identifiers.
  4. **Insights.** What the dream noticed — a step that keeps stalling, a question the person keeps being asked, a skill the process lacks — goes into dream memory.
- **Entering a dream** *(the operator's direction, 2026-09-25; the form is proposed)*. Nobody dreams by accident, and a dream never spends what the person did not allow.
  - **A dream policy per orchestrator**, the main loop's own included. Each process states its policy in `policy.dreams` (curator-playbook §6.8, §6.13): time windows, a per-night budget of tokens and minutes, and conditions that must all hold:
    - **idle:** no person attached to the session and no urgent message queued for it;
    - **capacity**, any one of these:
      - subscription quota remaining above a share;
      - quota that would expire unused before its reset (use it or lose it);
      - a local model engine that is ready or ensurable (curator-inference-manager);
    - **AC power;**
    - **within the per-night budget.**
  - **The main loop schedules the dreams** of every process on the node within the node's capacity: the operator's machine capacity (`~/.curator/runtimes.toml`), the provider-limit state and the engines available. It never dreams two processes on one quota when one would exhaust it.
  - **Approval once.** A process dreams only under a capability grant of the kind `dreams`, asked once per process (curator-trust §16), or given ahead with `keeper grant <hive> dreams`. A dream cycle then needs no touch: nothing in a dream carries authority, and each proposal it produces takes the ordinary batched confirmation.
  - **The sequence:**
    1. The session host asks the orchestrator to park, and the orchestrator writes its park note (session-host specification).
    2. The host stops the session.
    3. The host starts a dream session with the role's **dream profile** on the dream sandbox. The dream profile is a Curator profile variant of the role's profile. It carries its own system-prompt modules ("you are dreaming: nothing here reaches the world…"), the dream-mode tools, the log distiller and the consolidation skills.
    4. The dream writes consolidation into the memory ladder (§13.4), derived from real logs and within the memory budget. It also writes quarantined insights into dream memory and rehearsal reports.
    5. As the policy says, the host then either resumes the normal session, from its goal, its park note and a pointer to the dream's report, or leaves it parked.
  - **Waking.** A real high-priority doorbell wakes the dreamer: a message from a person, an escalation, an emergency. The host parks the dream in its own state and resumes the normal session at once.
- **The log distiller.** A separate deterministic tool, not a model, prepares the harness logs a dream reads. It removes noise — code blocks, diffs, long tool outputs, repeated content — and summarizes nothing, so what the dream reads are the sessions' own words with the noise taken out. It keeps the identifiers that link every passage back to level 0 of the ladder, the journal, so an insight can always point to where it came from. It also redacts secret-shaped content, such as keys, tokens and passwords, that leaked into a transcript (curator-trust §20).
- **Dream memory.** A separate store, quarantined until the person reviews it. The system roles read it and turn insights into proposals:
  - **the designer** proposes improvements to a playbook, which wait for a batched touch like any other change to a process (§9.6);
  - **the librarian** proposes consolidations of processes (§9.4);
  - **the sysadmin** proposes improvements to the apiary itself — its profiles, its skills, the main loop's own process — as change sets to the apiary's repository, applied only with the person's consent (§12.1).

  Nothing in dream memory changes a process or the apiary by itself, and nothing in it is an instruction to any session.
- **Review.** Dream-memory insights join the monthly review (§13.4) and the batched confirmations; an insight the person rejects is kept as rejected, so the next dream does not propose it again.

## 14. Dependencies

| Needs | From | Status |
| --- | --- | --- |
| start a hosted session in a directory, detached, with a profile and a goal; attach in control or read-only mode, for `keeper chat` and `keeper watch`; park, stop and the emergency stop; doorbells into detached sessions; a session registry for the picker; per-session capability tokens; session events and telemetry for the watchdog; its command line `curator session …` | the session host (its own specification; `ax` later) | extracting today's daemon comes first (the operator's priority, 2026-09-24); closing access to the host is the first refactor after it (decided 2026-09-25); the rest is new |
| `curator run` as the one entry point; environment profiles installed from a package directory, with machine overlays; source shorthands and default directories; `profile export` and `profile restore` of profile declarations; the attestation fast path and a signed bootstrap; commands built from source with the system's Go when it fits; the registry of audited skills; template packages instantiated without an upstream; command-only installs (`global add … --commands-only`), so `keeper` is on the person's `PATH` while the role stays in its profile; provider installs (`curator provider install|check`) and first-party providers beside `curator`, since `keeper up` needs `curator-run`; the operator-file commands `curator models` and `curator runtimes` | Curator (Decisions 0004, 0005, 0010, 0016, 0019, 0021; curator-spec issues #92, #93, #94, #95, #100; curator issues #87, #96; #91 closed) | profiles and overlays exist; the decisions are proposed; the issues are open |
| messages, doorbell, local and NATS carriers, leases, messenger providers, a companion-app signer provider | waggle (CM1, CM2, providers; Amendment A1) | CM1 is next; A1 is written |
| processes, hand-off, delegation semantics, skill policy, template parameters, dream mode; the validator `curator playbook check`; `task-board trust confirm` and `task-board config status\|propose` | curator-playbook | DRAFT v4.4 (the core) |
| who counts as a person, confirmations and their batching, capability grants, yolo, the companion app and pairing, secrets, separation at the operating-system level | relux-works/curator-trust `spec/trust.md` | DRAFT |
| the memory skill and its package-prefixed command | its own package, installed with the profiles | proposed (§13.4) |
| local engines and models for the first run, through `curator engines` | curator-inference-manager | specified |
| the board; the kernel commands keeper calls: `trust …`, `config status`, `mail …`, `board snapshot`, the directive kind `park` | task-board | the board exists; the commands are planned (curator-playbook, curator-trust, waggle, the session-host specification) |
| hardware requirements and download sizes for local models | the model registry (agents-management) or the Curator registry | open |
| a user-level login agent | the node's operating system (launchd, systemd) | exists |
| the companion app | its own track | not started |

## 15. Integration plan and open questions

### 15.1 Integration plan *(proposed; KP1 terminal-only decided 2026-09-28)*

KP1 is terminal-only: no channel and no tick. Channels come in KP2, and the tick with its user timer comes from curator-playbook PC3.

| Step | Delivers |
| --- | --- |
| KP0 | this draft, its diagrams, a refuter run |
| KP1 | one node on existing harnesses: the detached root and the keeper package's profiles, with the providers `curator-run` and `curator-session`; the `keeper` command — `keeper` itself, `init`, `up`, `down`, `session list` (`ls`), `chat` and its picker, `watch`, `park`, `stop`, `estop`, `status`, `harness`, `hive` (with `add --template`), `hint`, `route`, `ask`, `wake`, `host` — over `curator session` and `curator run`, and over today's `task-board session` commands until the node host lands; the login agent; the registry; supervision through the extracted session host with capability tokens; routing with a generative fallback and hints; creating a process from a template; the main loop's own process |
| KP2 | channels through waggle providers (`keeper channel`); the app's download link and the pairing QR code in the terminal; pairing with the companion app, with and without a screen (`keeper device`), and approvals, grants and batched confirmations through it (on the app's own track) and at the terminal (`keeper inbox`, `approve`, `reject`, `confirm`, `grant`, `revoke`) |
| KP3 | the designer's generation and improvement proposals, with dream rehearsals and the nightly dream cycle once curator-playbook dream mode ships; the librarian; skill discovery and installation; the memory skill with its resolution ladder, files backend first, and `keeper memory`; `keeper review` |
| KP4 | the sysadmin: backup (`keeper backup`), restore (`keeper restore`), the token inventory, the trusted-skills list, skill updates; the watchdog |
| KP5 | several nodes: joined processes, a head main loop, several operators |

### 15.2 Open questions

1. **The session host's attach protocol and picker registry**, and who may converse (`keeper chat`) rather than only watch (`keeper watch`).
2. **Doorbells into detached sessions:** at which safe point they are delivered, and the waggle §10 amendment.
3. **An operator hierarchy** across several nodes: who may confirm what, for whom.
4. **The head main loop's relation to other nodes' main loops:** which requests they accept and what they report.
5. **Hardware requirements for the first-run local model:** where they are published, and where download sizes and time estimates come from.
6. **Harness authentication through the chat**, with device codes relayed to the person.
7. **The companion app:** which platform comes first; the secure-storage abstraction on each platform; who runs the relay; push; SSH administration; losing or replacing the device.
8. **MCP as a second surface** for the module.
9. **Routing hints on a process worked from several nodes:** how they are shared between the operators.
10. **Routing thresholds:** the threshold and the margin that decide `route` against `ask`, and how they are tuned.
11. **Protecting the root directory, `keeper.toml` and the operator's files** from replacement by a process running as the operator: the open question on operating-system-level protection of configuration files (curator-playbook §17.3).
12. **Backup and restore:** what exactly is backed up (the operator's files in `~/.curator`, which hold no secrets?), whether the backup is encrypted at rest, how a restore is verified, and partial restores of one process.
13. **Moving secrets:** the protocol between the app and a node, keychain formats on each platform, and rotation.
14. **Trusted-skills certificates:** their format, who issues them (Curator's audit), how they are revoked.
15. **Headless pairing:** the length of the pairing code and the authentication string, trust in pairing relayed over SSH, and the tailnet bootstrap.
16. **Capability tokens before CM2:** format, issuance, revocation, and how the operator's command line proves presence for the emergency stop.
17. **The emergency stop:** which harnesses can stop a run so that it resumes, and what the snapshot contains beyond the board.
18. **Watchdog telemetry:** which signals each harness exposes (context usage, compaction hooks, token events) and how the session host passes them on.
19. **Profiles:** the format of the package index that names a repository's profiles and its default (curator-spec issue #92).
20. **Personal memory:** whether accepted notes need each person's review, or only quarantined ones (§13.4).
21. *Resolved on 2026-09-25:* on-demand skills are withdrawn and curator-spec issue #91 is closed; role skills reach their runs through role profiles (§13.5). The item keeps its number.
22. **The memory skill:** the size of the daily write budget, the quarantine rules, and the search backend beyond files.
23. **The phone's permission prompts:** how grants are worded and grouped, how many confirmations seal a grant, and how yolo is shown (curator-trust).
24. **The memory ladder:** the size bounds of each level and of the fact cards, when day, week and month summaries are written (at the first session of the next period, or on a schedule of the main loop's process), and the identifier format (§13.4).
25. **Dream rehearsals:** which requests the designer rehearses by default, how long a rehearsal may run, and how its report is shown in the confirmation request (§9.2; curator-playbook §17.3 item 19).
26. **The nightly cycle and dream memory:** the defaults of `policy.dreams` and how the main loop orders processes that compete for one night's capacity (the form is proposed in §13.6); the log distiller's rules and output format, and how its identifiers map onto level 0; how dream-memory insights are grouped for review and how long rejected ones are remembered (§13.6).
27. **`keeper ask`:** how the answer reaches a script when the request needs a person or takes long — exit codes, `--json`, a timeout, a later `keeper inbox` entry (§7.2).
28. **The memory package's command name:** its package prefix, and whether `keeper memory` stays a pure front or the main loop's library calls the package directly (§13.4).
29. **A harness per hive:** `[bindings]` in `models.toml` has no per-project key, so two hives of one playbook share their root role's binding; whether bindings gain a `projects` scope, as standing answers have (curator-playbook §2.3), or keeper keeps a per-hive preference in the operator files (§7.4).

## 16. Change log: v0 → v0.1

Applied from the operator's answers of 2026-09-25:

| Change | Where |
| --- | --- |
| The approved beekeeping names open the specification; command lines, directories and profiles use them; the pending-names section and its open question are gone | Names, §2 |
| Principles: the journal is the state; authority only narrows; transport is not authority; privileged operations outside agent shells; the emergency stop in the authority plane | §1.1 |
| Installation detached from our remotes: the root is instantiated from a pinned, audited template package; updates only through the Curator registry, proposed by the main loop or a root orchestrator | §6.1 |
| One root package installs everything; one board tool for many boards | §6.2, §5.1 |
| Commands built from source with a pinned Go toolchain; signed binaries later | §6.3 |
| First run: branch A states the download before starting; branch B is semi-automatic; pairing with and without a screen; the login agent | §6.4, §11.3 |
| `keeper up`, `down`, `ls`, `attach` with a picker, `park`, `stop`, `estop`, `status`; `keeper start` is replaced by `keeper up` | §7.2, §7.5 |
| The launch nesting from `curator` to the harness | §7.3 |
| Any harness; the default is the first one set up; overrides | §7.4 |
| Who may act on which session, before and after CM2; the operator's command line | §10.2 |
| The emergency stop | §10.3 |
| `keeper up` as the single entry point after a reboot, and its order | §10.6 |
| The companion app over one secure-storage abstraction, on every platform; a Mac as the control panel; transport through an abstraction; an SSH shell later | §11.3 |
| The sysadmin: backup to the person's own remotes, secrets outside git, restore, the trusted-skills list, the token inventory, skill updates | §12.1 |
| The watchdog: context, compaction and provider limits | §12.2 |
| Memory, persona and overlays; profiles as package content, open | §13 |
| New open questions 11–20 | §15.2 |

## 17. Change log: v0.1 → v0.2

Applied from the operator's answers of 2026-09-25:

| Change | Where |
| --- | --- |
| `keeper` with no arguments opens the conversation with the main loop, starting the apiary first when it is down; `keeper chat [hive]` replaces `attach`; `keeper watch <target>` is the read-only view; `keeper harness default` | §0, §7.2, §7.5, §10.2, §11.1 |
| The default harness is recorded in `keeper.toml`; when no logged-in harness is found, `keeper init` asks which installed one becomes the default | §6.4, §7.4 |
| Installation: the system's git or a lightweight one, never the full Xcode; the system's Go, never overridden; a signed bootstrap and the attestation fast path instead of a long local audit; source shorthands and default directories | §6.3, §6.4; curator-spec #92, #94; curator #87 |
| One install brings everything: the keeper package's `apiary` profile is the install unit, and `keeper init` adds the system roles' profiles; the declarations are exported into the root | §6.2, §5.1; curator-spec #93 |
| Pairing: first the app's download link as a QR code in the terminal after choosing iOS or Android, then the pairing QR code, each with a code as fallback | §6.4, §11.3 |
| Privileged operations named; agents request them and never hold them | §1.1 principle 4 |
| Permissions like a phone's, batched touches for structure changes, yolo mode | §0, §9.6, §11.5 |
| Backup into one repository of the person's: `main`, `hive/<name>` branches, `mirror/`, `curator/`; restore re-points sources to the mirror; Curator verifies signatures instead of auditing again | §12.1 |
| Memory as a plug-and-play skill with provenance, a daily write budget, quarantine and a notes digest; the persona as a machine overlay; profiles as the install unit; role profiles instead of project folders | §13 |
| The trust material now lives in relux-works/curator-trust; curator-playbook is referenced as its DRAFT v4.0 core | Related documents, §14 |
| New open questions 21–23 | §15.2 |

## 18. Change log: v0.2 → v0.3

Applied from the operator's discussion of 2026-09-25:

| Change | Where |
| --- | --- |
| References move to curator-playbook DRAFT v4.2 (the core) | Related documents, §9.6, §14 |
| The `keeper` command on the person's `PATH` through a command-only install, so the role stays in the `apiary` profile (curator-spec #95) | §6.2 |
| Memory as a resolution ladder: the journal, session, day, and week or month summaries, and bounded fact cards; identifiers on every line; `memory expand` and `memory search`; a snapshot at session start; capacity errors that force consolidation; optional write approval | §13.1, §13.2, §13.4 |
| Role profiles decided; on-demand skills withdrawn (curator-spec #91 and curator #91 closed) | §13.5, §14, §15.2 item 21 |
| Dream rehearsals by the designer before it asks for enablement, with the report in the confirmation request | §9.2, §9.3 |
| The keeper skill only in the `apiary` profile and the `keeper` command global, decided (curator-spec #95, curator #95; #91 closed) | §6.2 |
| Dreams as the nightly cycle: replay of distilled logs through the deterministic log distiller, rehearsal, consolidation into the ladder, and a quarantined dream memory that the designer, librarian and sysadmin turn into proposals — playbooks under a batched touch, apiary change sets with consent | §13.6 |
| New open questions 24–26 | §15.2 |

## 19. Change log: v0.3 → v0.4

Applied from the operator's answers of 2026-09-25 to v0.3:

| Change | Where |
| --- | --- |
| References move to curator-playbook DRAFT v4.3, which makes loop limits template parameters a project tunes down to unbounded | Related documents, §9.6, §14 |
| Entering a dream, proposed: a dream policy per orchestrator, the main loop included (windows, a per-night budget; the conditions idle, capacity, AC power and budget, where capacity is quota above a share, quota that would expire unused, or a ready or ensurable local engine) | §13.6 |
| The main loop schedules dreams across processes within the node's capacity; approval once per process through the capability kind `dreams` | §13.6 |
| The dream sequence: park, stop, a dream session with the role's dream profile on the sandbox, consolidation, insights and reports, then resume or stay parked; a real high-priority doorbell wakes the dreamer | §13.6 |
| Open question 26 narrowed to the policy defaults and to ordering processes that compete for one night's capacity | §15.2 |

## 20. Change log: v0.4 → v0.5

Applied from the operator's adoption, on 2026-09-27, of the command-line shape in wiki `research/cli-shaping-review.md`:

| Change | Where |
| --- | --- |
| Beekeeping words appear only in `keeper`'s nouns, the apiary's directories and the profile name `apiary`; every kernel command, flag, configuration key and provider uses the technical terms | Names |
| Three front doors: `keeper` for the person, `curator` with its `curator-<noun>` providers for the machine, `task-board` as the kernel; keeper wraps the other two | §0, §2, §7.2 |
| The keeper command map, with a column naming what each command calls: `init`, `restore`, `up`, `down`, `status`, `chat`, `watch`, `wake`, `park`, `stop`, `estop [<hive>]`, `route [--dry-run]`, `ask`, `inbox`, `approve`, `reject`, `confirm`, `grant`, `revoke`, `review`, `session list` (alias `ls`), `hive list\|show\|add\|remove\|set`, `hint`, `channel`, `device`, `harness list\|set` (alias `harness default`), `memory`, `backup`, `host`; `keeper create` folds into `hive add --template\|--design\|--path\|--clone`, and `hive move` into `hive set` | §5.4, §7.2 |
| The default harness is the operator's `@orchestrator` binding in `~/.curator/models.toml`, not a key of the committed `keeper.toml`; a hive's harness is the operator's binding for its root role; runtime ids everywhere | §5.1, §5.2, §5.4, §6.4, §7.4 |
| Curator's providers are installed by `keeper init` (`curator provider install`, curator-spec #100, curator #96), because no profile can carry them; the command-only install is `curator global add … --commands-only` (#95); the declarations are exported and restored with `profile export` and `profile restore` (#93) | §6.2, §6.4, §7.3, §12.1, §13.3, §14 |
| `keeper init` spelled as its steps: the role profiles, the providers, `task-board init`, `task-board trust init`, the first commit, pairing and the login agent | §6.4 |
| `keeper up` runs `curator run <runtime> --profile <role profile>` for each hive | §7.3 |
| Session commands wrap the session host's `curator session …`; `task-board session` stays an alias for one release after the node host lands | §2, §7.2, §7.5, §10.2 |
| One emergency stop, `keeper estop [<hive>]`, over `curator session estop` and the kernel's `task-board board snapshot`; runs park by the directive kind `park` | §10.3, §10.5 |
| The person's decisions at the terminal: `keeper inbox`, `approve\|reject`, `confirm`, `grant\|revoke` (yolo and dreams are kinds), `review` and `device`, over `task-board trust …` | §11.1–§11.5, §13.6 |
| The memory package's command takes its package's prefix, with `keeper memory` in front; the validator is `curator playbook check`; the per-role Curator profile is `spawn --profile` and `spawn.profile` | §6.2, §9.2, §13.4, §13.5 |
| References move to curator-playbook DRAFT v4.4, which renames its commands to the same shape | Related documents, §9.6, §14 |
| New open questions 27–29 | §15.2 |

## 21. Change log: v0.5 → v0.5.1

Applied from the operator's answers of 2026-09-28:

| Change | Where |
| --- | --- |
| Our own loop is postponed; the main loop runs on the existing harnesses, and harness abstraction is named as a core advantage and a selling point; replaces the 2026-09-24 wording | §7.7 |
| KP1 is terminal-only: no channel and no tick | §15.1 |
