# Main loop: specification

Status: **DRAFT v0**, 2026-09-24. Not normative. Repository `curator-keeper`, after the component's working name (§2.1). Track: [`track.md`](track.md). Diagrams: [`../diagrams/`](../diagrams/). Earlier research: wiki `research/keeper-and-process-lifecycle.md` and `research/agent-loop.md`; roadmap line KP (`wiki/roadmap/ecosystem-roadmap.md` §5).

One person runs several processes on one machine: a family calendar, the smart home, a hobby project, a small café. Each process has its own playbook, board and root orchestrator (curator-playbook `spec/process-configuration.md`). Something has to sit above them. It must:

- know which processes exist;
- take a request in plain words, such as "order oat milk", and decide which process it belongs to;
- start that process's orchestrator when it is not running, and wake it when a message arrives;
- bring results back to the person;
- ask the person what only a person may decide.

This specification describes that component, the **main loop**: precisely for one machine, and for several machines as far as the architecture must allow now.

Markers:
- *decided (2026-09-24)*: the operator decided it;
- *proposed*: the author's proposal;
- **pending operator decision**: named where it applies.

Related documents:
- curator-playbook `spec/process-configuration.md`, DRAFT v3.3: processes, root orchestrators, hand-off, delegation, the operator's files and their confirmation;
- waggle `spec/waggle.md` with Amendment A1: messages, doorbell, carriers, leases, board acts;
- the session-host module's specification (working name `agent-session-host`);
- curator-spec Decisions 0010 (environment profiles), 0013 and 0019 (launch plans, one construction site) and 0021 (sessions enter through `curator run`, hosted through the `ax` contract);
- the agent-session-manager (`ax`) specification;
- curator-inference-manager `spec/inference-plane.md`.

---

## 0. The design on one screen

- **One main loop per machine.** It manages the machine's **processes** — each a project directory with its own `Playbook.json`, board and exactly one **root orchestrator** — and does no process work itself. *proposed; one root orchestrator per process, started from outside, decided (2026-09-24)*
- **The session host keeps orchestrators alive.** The main loop starts a process's root orchestrator through the session host, which keeps it running in the background. The root orchestrator then runs its own process and spawns sub-orchestrators and runs inside it. *decided (2026-09-24)*
- **The main loop and orchestrators talk over waggle.** A request goes in as a message, the session host rings the doorbell, and the orchestrator pulls the message; reports come back the same way. Child runs never talk to the main loop. *proposed*
- **A registry points at separate repositories.** The main loop keeps a **process registry**, `keeper.toml`, in its own **root directory**. Processes are separate repositories the registry points at, never git submodules. Adding, removing or moving a process is authority-bearing and takes effect after a person confirms it. *proposed*
- **Routing** follows a fixed order *(decided 2026-09-24)*:
  1. an explicit reference wins;
  2. then a channel bound to a process;
  3. then one confident candidate;
  4. close candidates get one short question, and the answer becomes a visible routing hint;
  5. no candidate leads to the process designer;
  6. a request for two processes is split.

  A fast classifier with typed outputs fits; the specification does not prescribe one.
- **The main loop is a role plus a module.** The role is a skill any harness can run; the module is the `keeper` command and library that does the deterministic work. The minimal system runs entirely on existing harnesses. Our own lightweight loop is optional and later. *proposed*
- **Processes are created, not configured by hand.** The main loop matches a request against published templates first. When none fits, the **process designer** generates a playbook, always starting from the simplest process that works. The **librarian**, a small process of its own on a timer, proposes merging near-duplicates and merges only after the person approves. *decided (2026-09-24)*
- **The person talks and approves remotely.** Conversation goes through channels: the terminal first, then messengers through waggle providers. Approvals go through a **companion app** on the phone, the common case, or through the operating system's user authentication on the machine. *decided (2026-09-24)*
- **Several machines are architected for, not implemented now:** one process worked from several machines under leases, unrelated processes on several machines under a head main loop, several operators, and external participants. *proposed*

## 1. Goals and non-goals

Goals:

1. A person reaches every process on the machine through one conversation, without knowing how the processes are organized.
2. A new kind of work becomes a new process without hand-editing configuration.
3. Background orchestrators run autonomously and are observable on demand.
4. What only a person may decide reaches the person wherever they are, and the answer carries proof that the person gave it.
5. Nothing in the one-machine design blocks several machines, several operators or external participants later.

Non-goals of this draft:
- the process model (curator-playbook);
- message formats, signatures and carriers (waggle);
- how sessions are hosted (the session-host specification);
- our own agent loop (wiki `research/agent-loop.md`);
- the companion app's implementation (§11.3 lists what the main loop needs from it).

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
| session host | the node's component that starts, keeps and wakes long-lived agent sessions (the session-host specification; today task-board's session daemon `tb-sessiond`) |
| operator | a person who administers the node and answers what only a person may decide |
| external participant | an agent of another owner that works on one of our processes under a lease and signatures |
| companion app | the operator's phone app, which renders requests for approval and signs them (§11.3) |

### 2.1 User-facing names — pending operator decision

User-facing material uses Apiary names; specifications, code and command lines use the technical terms above. The mapping in the wiki (`naming.md`, 2026-09-24) makes a Hive one node and a Comb one process. On 2026-09-24 the operator found that mapping hard to follow. The proposal below follows the beekeeping picture instead:

| Name | Technical term | What it is |
| --- | --- | --- |
| **apiary** | the whole system | every node, process and operator of one owner; one node is a small apiary |
| **keeper** | main loop | tends the apiary: one per node, and one head keeper when there are several |
| **hive** | process | one domain: family calendar, smart home, café, a hobby project |
| **queen** | root orchestrator | exactly one per hive |
| **comb** | board | its cells are the tasks |
| **bee** | run | an agent at work |
| **wild bees** | external participants | agents of another owner working on one of our hives under leases and signatures |
| **nuc** | process package | the starter colony a new hive grows from: a published playbook |
| **yard** | node | one machine of the apiary |

This is the only section of the specification that uses these names. Four working identifiers of this draft already take the proposed names and will follow the decision:
- the repository and command `keeper`;
- the default root directory `~/apiary`, with processes under `hives/`;
- the Curator environment profile `apiary`;
- the command noun `hive` in `keeper hive …`.

If the operator keeps technical identifiers, the noun becomes `process` and the profile gets a technical name.

## 3. One node

### 3.1 Who does what

| Component | Does | Does not |
| --- | --- | --- |
| main loop | keeps the registry; takes the person's requests and routes them; creates processes through the designer; starts, wakes, parks and stops root orchestrators; relays results and questions; runs its own small process (§5.1) | write any process's board; spawn a run inside a process; change a process's `Playbook.json` except by a committed change the person confirms |
| session host | starts sessions in a given directory with a given profile and goal; keeps them alive while detached; lets people attach; rings the doorbell; emits events | decide what a session does |
| root orchestrator | runs its process: plans, spawns sub-orchestrators and runs, integrates, asks for people through the main loop | write another process's board |
| sub-orchestrators, runs | the work inside one process, under the root orchestrator's leases | talk to the main loop |
| operator | answers and approves; confirms authority-bearing changes | edit configuration by hand (possible, never required) |

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
3. **Several operators.** Several people administer one system, possibly in a hierarchy. Operators are waggle principals (waggle §3), and who may confirm what is policy (curator-playbook §2.3 `second-person`, §4.2). An operator hierarchy is an open question (§14).
4. **External participants.** Agents of another owner work on one of our processes under a lease and signatures, admitted by that process's roster (waggle milestone CM4).

Three choices now keep all four ways open without a redesign:
- every registry entry names the process's owning node and a stable process id (§5.2);
- every message between a main loop and an orchestrator is a waggle message, even on one node, so crossing machines changes only the carrier (§10.3);
- processes are separate repositories, so a process can move to another node or be cloned there (§5.3).

![Several nodes](../diagrams/artefacts/plantuml/several-nodes.svg)

## 5. The root directory

### 5.1 Layout *(proposed)*

```
~/apiary/                      the main loop's root: a git repository
  Skillfile.json               the keeper skill, the designer and librarian roles, the board
  Playbook.json                the main loop's own process: requests, routing, "a person is needed", the librarian's runs
  keeper.toml                  the process registry (§5.2)
  task-board.config.json       where the main loop's board lives
  .task-board/                 the main loop's board: requests and what became of them
  .gitignore                   ignores hives/
  hives/                       default place for process directories, each its own repository
    family-calendar/           Playbook.json, Skillfile.json, task-board.config.json, .task-board/
    cafe/
```

- **The main loop has a process of its own.** Its elements are the person's requests and the questions the main loop asks, and its workflows track each request from arrival to answer. So the main loop's work is visible on a board like any other process's, and the librarian can run on a schedule (§9.4).
- **Process directories may live anywhere.** `~/work/relux-app` is as good as `~/apiary/hives/relux-app`; the registry records the path.

### 5.2 The process registry *(proposed)*

```toml
[processes.family-calendar]
id          = "proc-7f3a2c"                  # stable, never reused
path        = "hives/family-calendar"        # relative to the root, or absolute
remote      = "git@github.com:example/family-calendar.git"   # optional
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
- `description` is what routing reads, so it says what the process handles in the person's words. The designer writes it and the person may edit it.
- The registry holds declarations only. Which sessions run, and in which state, is runtime state kept beside the registry, never committed.

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

  They follow the rule curator-playbook applies to its authority-bearing sections. The main loop may write them, as a commit to its root repository, but they take effect only after a person confirms the new version (§11.2).
- **Which are not.** Descriptions and routing hints only choose where a request is offered; every process still applies its own rules. The main loop writes hints from the person's answers (§8.3), and they take effect at once.

## 6. Installation from zero *(proposed; the flow was decided in outline on 2026-09-24)*

1. **Curator.** A script from the website, or Homebrew, installs Curator, the way agent vendors install their harnesses.
2. **The root.** Curator initializes the root directory from a template package, a small repository published with the keeper skill. The template holds `Skillfile.json`, the main loop's `Playbook.json`, an empty `keeper.toml` and `task-board.config.json`. Curator then installs:
   - the keeper skill, which exports the `keeper` command, built by Curator from the skill's source as it builds other commands;
   - the board (task-board), as a runtime dependency of the orchestrator skills;
   - the process-designer and librarian roles, on demand (curator-playbook §8.1);
   - the base Curator environment profiles (Decision 0010), among them the main loop's profile `apiary` with its root context and skill set.

   Finally it initializes the main loop's board and commits the root.
3. **A model to talk with.** The main loop needs a model before it can talk. Curator checks the hardware against the minimum requirements published with the model registry, and the installation takes one of two branches:
   - **(A) The hardware suffices.** Curator installs an inference engine package and a small local model through curator-inference-manager (`~/.curator/engines.toml`), and the main loop starts on it at once. It greets the person with a short questionnaire: connect subscriptions? which harnesses (Claude Code, Codex, Muse, others)? Each yes becomes a runtime binding in `~/.curator/runtimes.toml`, and the person logs in to that harness.
   - **(B) It does not.** The first mandatory step is to authenticate a harness fit for the root-orchestrator role, and the main loop starts on it.

   Later, harnesses may be authenticated through the chat, with the device codes a harness prints relayed to the person. The first versions log in by hand.
4. **Channels.** The main loop offers to connect messengers (Telegram, WhatsApp and others) through waggle providers, so the person can write to it from the phone (§11.1). Until then the terminal is the channel: `keeper attach`.
5. **The companion app.** The main loop shows a pairing code, as a QR code in the terminal or on a page, and the person scans it with the companion app. The node and the phone exchange and pin each other's keys (§11.3). From then on, approvals, delegations and confirmations reach the person anywhere.
6. **The first process.** The person asks: "what can you do?", "keep our family calendar". The main loop proposes processes from the published packages, and the designer creates the chosen one (§9).

What exists today: Curator, `curator run`, task-board with its session daemon, and `task-board spawn`. Everything else in this list is planned.

![Installation and first run](../diagrams/artefacts/plantuml/installation-and-first-run.svg)

## 7. The main loop: a role plus a module

### 7.1 The role *(proposed)*

The main loop's session runs a harness — Claude Code, Codex, or any other with a session-host adapter — with the keeper skill as its identity. The skill arrives through Curator's skills launch channel (curator-playbook §8.1). It says:
- how to talk with the person, and when to ask and when not to;
- how to route, and when to create a process;
- which `keeper` command does each step.

The model decides; the command does.

### 7.2 The module *(proposed)*

The `keeper` command and its library do the deterministic work, with `--json` output for agents and people alike:

| Command | Does |
| --- | --- |
| `keeper hive list`, `keeper hive show <process>` | the registry, with each process's state: running, parked, stopped, unreachable |
| `keeper hive add`, `remove`, `move` | proposes a registry change, which takes effect after confirmation (§5.4) |
| `keeper route "<text>" [--channel C] [--from P]` | the candidate processes with scores and reasons (§8) |
| `keeper create --template <package>` or `--design "<text>"` | creates a process (§9.3) |
| `keeper wake <process> [--message …]` | starts or wakes the root orchestrator and delivers a message (§10) |
| `keeper park <process>`, `keeper stop <process>` | parks or stops a root orchestrator through the session host (§10.2) |
| `keeper status` | the node: processes, sessions, questions pending for the person |
| `keeper start` | starts the main loop's own session in the background |
| `keeper attach [<process>]` | attaches the terminal to the main loop's session, or to a process's root orchestrator |

- **`keeper start`** runs `curator run` with the `apiary` profile in the root directory. The session is hosted from birth by the session host through the `ax` contract (Decision 0021; today task-board's session daemon bridges it) and starts detached.
- **`keeper attach`** shows that session's harness exactly as it runs, the Claude Code or Codex interface, so the person can watch it and type into it. Detaching leaves it running.

### 7.3 Skills first *(proposed; the operator's direction on 2026-09-24, not a decision)*

The main loop's capabilities reach a harness as skills installed through Curator: a skill appears in every harness Curator manages, and profiles carry it into launches. An MCP server needs configuration inside each harness. Curator may manage that later (Decision 0010 lists MCP among future profile surfaces), and the module can serve both.

Not every role has a command:
- the board is a command the orchestrator skills depend on;
- the keeper skill exports `keeper`;
- most process roles have none.

### 7.4 Our own loop later *(proposed)*

The minimal system runs entirely on existing harnesses. Our own lightweight loop (wiki `research/agent-loop.md`) becomes worth building for three things, none of them required now:
- an always-on front on a small local model, cheaper than a full harness session kept idle; a harness bound to a local engine through a profile and curator-inference-manager already gets most of that;
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

- **It always exists.** It is installed with the root (§6).
- **It writes a `Playbook.json` when no template fits**, extending the closest template where there is one. The validator checks it (`playbook validate`), and the refuter may challenge it.
- **It starts from the simplest process that works:** the fewest states, roles and fields that carry the request. It adds complexity only on evidence: repeated escalations, recurring manual steps, reviews that keep failing. *decided (2026-09-24)*
- **It keeps processes alive.** The main loop measures each process: throughput, rework, time spent waiting on people, escalations. The designer proposes improvements the way a bug on a board is fixed: a change to the process's `Playbook.json`, validated, then confirmed (§9.6). *decided (2026-09-24)*

### 9.3 Creating a process

1. The designer drafts the project directory: `Playbook.json`, `Skillfile.json`, `task-board.config.json`, a board, and a description for the registry.
2. The draft passes the validator, and the skills the process needs are installed (§9.5).
3. The draft is committed in the new repository, and the registry entry is written.
4. The person confirms (§11.2). One confirmation covers the process's authority-bearing sections and the registry entry together.
5. The main loop starts the root orchestrator (§10.1) and hands it the request that led here.

### 9.4 The librarian

- **A small process of its own**, on a schedule (daily or every two days) in the main loop's `Playbook.json` (curator-playbook §6.6).
- **Finds near-duplicates.** It compares the node's playbooks by structure (the semantic digests and structure of their resolved workflows) and by meaning.
- **Drafts a generalized template.** What the duplicates share becomes a template they `extend`, their differences become slots and fields, and running elements get migrations.
- **Merges only after approval.** The main loop proposes the consolidation to the person, and the librarian merges only after the person approves. *decided (2026-09-24)*

### 9.5 Skills a process needs

- **Reuse before writing.** The designer and the orchestrators look for existing skills before writing their own: a CRM integration for a café, a supplier's portal, a calendar provider. The source is the Curator registry, whose skills pass Curator's audit.
- **Audited skills install autonomously when allowed.** An audited skill from a trusted registry installs without asking when policy or a delegation allows it; otherwise the person confirms it.
- **Skills from unknown sources are refused by default,** like software from unidentified developers on macOS. Only a deliberate operator setting that states the risk enables them. The recommended path is to submit the skill for audit, through the main loop, so that it enters the Curator registry. *decided (2026-09-24)*
- **Using a skill's tool stays the process's decision.** The tool is bound and enabled in the process's `Playbook.json` (curator-playbook §8.5).

### 9.6 Changing a process *(decided 2026-09-24)*

- A change to a process's rules is a commit to the default branch of the process's repository (curator-playbook §2.4). No pull request and no hosting service is required; a project that uses one may keep it.
- Authority-bearing sections take effect only after a person confirms the committed version (§11.2). The other sections take effect on commit, and an orchestrator may change them under delegation.
- A process without git applies a new version only through the same confirmation.
- curator-playbook takes the confirmation rule for authority-bearing sections in its next revision.

## 10. Supervision and communication

### 10.1 Starting a root orchestrator *(proposed)*

- **Start.** `keeper wake` on a process with no running root orchestrator asks the session host to start one: `curator run` in the process's directory with the process's profile, detached, bound to the process's primary goal.
- **No second start.** The board's single primary-goal slot refuses a second one.
- **Record.** The main loop records the session and its state in its runtime state, not in `keeper.toml`.

### 10.2 Powers *(proposed)*

- **Four powers.** Over the root orchestrators it starts, the main loop keeps wake, park, stop and status. Parking means the session writes down where it stopped, and the session ends (session-host specification).
- **No micromanagement.** The main loop never directs work inside a process, never writes the process's board and never spawns into it.
- **Signed commands.** Park and stop reach an orchestrator as signed waggle `cmd` messages from the main loop's session, in the `halts` category that a process's signature policy may cover (curator-playbook §4.2). The session host executes them, and the orchestrator's model is only informed (waggle §5).
- **The person can always stop a session** directly through the session host.

### 10.3 Messages *(proposed)*

- **Always waggle.** Every exchange between the main loop and an orchestrator is a waggle message, on one node too: the local carrier on one node, NATS across nodes (waggle §9.3).
- **Who sends what.** A request for a process is a `coord.request` from the main loop's session to the root orchestrator. A question for the person, a result or a status goes back as `coord` from the orchestrator to the main loop. Child runs send `report` messages to their orchestrator only.
- **Delivery** follows waggle §10: the host appends the message to the mailbox, the session host rings a fixed-template doorbell, and the orchestrator pulls with `task-board mail read`.
- **Detached sessions must receive doorbells.**
  - Today task-board's notice pump delivers only to a session with an attached client (skill-project-management `docs/claude-goal-runtime.md`, "In-session run notices"), and waggle §10 lets messages wait while no client is attached.
  - Background root orchestrators are detached by design. So the session host must deliver doorbells into sessions nobody is watching, at a safe point of the turn.
  - This is a requirement on the session host and an amendment to waggle §10 step 4.

### 10.4 Work between processes

The main loop does not relay work between processes. Processes hand work to each other directly: by board records on one board, and by waggle across boards (curator-playbook §6.5). The main loop learns of it from reports when a person must be involved.

## 11. The person

### 11.1 Channels *(proposed)*

- **The terminal first:** `keeper attach`.
- **Messengers through waggle providers** (Telegram, WhatsApp, others). They are consoles and carriers that are untrusted pipes (waggle §9). What the person writes there enters as low-authority `coord`: it can ask and answer, never approve.
- **A channel may be bound to a process** (§5.2); everything written there routes to that process.

### 11.2 Approvals and confirmations *(decided 2026-09-24)*

Some answers need proof that the person gave them:
- approvals and relaxations that a process marks;
- delegations (curator-playbook §4.2);
- confirmations of the operator's files (curator-playbook §2.3);
- authority-bearing changes of a process or of the registry (§5.4, §9.6).

They go through one of two paths:
- **the companion app on the phone**, the common case, since nodes may be headless and the person is rarely at the machine;
- **the operating system's user authentication on the machine**: Touch ID or the password, Windows Hello, polkit or PAM, or a FIDO2 key.

The main loop presents the request, says what it is for, and waits. The answer is recorded with its signature.

### 11.3 The companion app *(proposed; its own specification comes later)*

What the main loop needs from it:

- **Pairing.** At installation the node shows a pairing code, and the app scans it. Node and phone exchange public keys over that out-of-band channel and pin each other, so no server in between can substitute a key.
- **Keys.** The app's signing key lives in the phone's hardware (the iPhone's Secure Enclave, Android's hardware-backed keystore), behind biometrics, so each signature needs the person's face or fingerprint. The app's attestation shows that the key is hardware-bound and the app is genuine. The key enrolls as the operator's key in the waggle roster; waggle §4.5 gains a companion-app signer provider.
- **What you see is what you sign.** The app renders the payload before signing (waggle §4.2).
- **Transport.** Requests and signed answers travel over an authenticated channel that resists man-in-the-middle attacks: directly over a tailnet when node and phone share one; otherwise through a relay that sees only end-to-end-encrypted envelopes addressed to the pinned keys, with push notifications to wake the app.
- **Administration.** The app may also carry SSH access to the node (open question, §14).

### 11.4 Standing answers under review *(decided 2026-09-24)*

The operator's files may hold open-ended standing answers (curator-playbook §2.3, §9). Once a month the main loop lists every open-ended answer, with when it last applied and how often, and asks the person to keep or withdraw each one. A kept answer is recorded as reviewed.

## 12. Dependencies

| Needs | From | Status |
| --- | --- | --- |
| start a hosted session in a directory, detached, with a profile and a goal; attach and detach; park and stop; doorbells into detached sessions; session events | the session host (its own specification; `ax` later) | extracting today's daemon comes first (the operator's priority, 2026-09-24); doorbells into detached sessions are new |
| `curator run` as the one entry point; environment profiles; on-demand skills and the skills launch channel; skill commands built from source; the registry of audited skills | Curator (Decisions 0010, 0019, 0021; curator-spec issue #91) | the decisions are proposed; #91 is open |
| messages, doorbell, local and NATS carriers, leases, messenger providers, a companion-app signer provider | waggle (CM1, CM2, providers; Amendment A1) | CM1 is next; A1 is written |
| processes, their confirmation rules, hand-off, delegation | curator-playbook | DRAFT v3.3 |
| local engines and models for the first run | curator-inference-manager | specified |
| the board | task-board | exists |
| hardware requirements for local models | the model registry (agents-management) or the Curator registry | open |
| the companion app | its own track | not started |

## 13. Integration plan *(proposed)*

| Step | Delivers |
| --- | --- |
| KP0 | this draft, its diagrams, a refuter run |
| KP1 | one node on existing harnesses: the keeper skill and the `keeper` command; the registry; `keeper start` and `keeper attach` through the extracted session host; starting and waking root orchestrators; routing with a generative fallback and hints; creating a process from a template; the main loop's own process |
| KP2 | channels through waggle providers; pairing with the companion app and approvals through it (on the app's own track) |
| KP3 | the designer's generation and improvement proposals; the librarian; skill discovery and installation |
| KP4 | several nodes: joined processes, a head main loop, several operators |

## 14. Open questions

1. **User-facing names** (§2.1), pending the operator's decision, and with them the working identifiers `keeper`, `~/apiary`, `hives/` and `apiary`.
2. **The Muse architecture note** the operator will send: which of its features affect this design now.
3. **The session host's attach protocol**, and who may attach with control rather than as a viewer.
4. **Doorbells into detached sessions:** at which safe point they are delivered, and the waggle §10 amendment.
5. **An operator hierarchy** across several nodes: who may confirm what, for whom.
6. **The head main loop's relation to other nodes' main loops:** which requests they accept and what they report.
7. **Hardware requirements for the first-run local model:** where they are published and how they change.
8. **Harness authentication through the chat**, with device codes relayed to the person.
9. **The companion app:** which platform comes first (iOS), who runs the relay, push, SSH administration, and losing or replacing the phone.
10. **MCP as a second surface** for the module.
11. **Routing hints on a process worked from several nodes:** how they are shared between the operators.
12. **Routing thresholds:** the threshold and the margin that decide `route` against `ask`, and how they are tuned.
13. **Protecting the root directory and `keeper.toml`** from replacement by a process running as the operator. This is the open question on OS-level protection of configuration files (curator-playbook §17.3).
