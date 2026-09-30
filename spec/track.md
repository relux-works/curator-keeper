# Track: main loop

Status: **DRAFT v0.5.1**, 2026-09-28. Specification: [`keeper.md`](keeper.md). Roadmap: `wiki/roadmap/ecosystem-roadmap.md`, line KP. Names: the keeper of an apiary (specification, Names).

## Why

The process configuration (curator-playbook) lets one project describe how it works. One person runs several such processes on one machine — a family calendar, the smart home, a hobby project, a café — and needs one place that knows them all. That place takes a request in plain words, sends it to the right process, keeps that process's orchestrator running, brings back results and questions, and brings the whole machine back after a reboot. Without it, the person has to open the right orchestrator by hand and remember how everything is organized.

## Milestones

| # | Delivers | Done when |
| --- | --- | --- |
| KP0 | draft specification, diagrams, a refuter run | merged as a draft; every refuter finding resolved or listed as open |
| KP1 | one node on existing harnesses, terminal-only (no channel and no tick): the root instantiated from the template package with no upstream; the keeper package's `apiary` profile as the one install, with the Curator providers `curator-run` and `curator-session` installed by `keeper init`; the `keeper` command (`keeper` itself, `init`, `up`, `down`, `session list` with its alias `ls`, `chat` with its picker, `watch`, `park`, `stop`, `estop`, `status`, `harness`, `hive` with `add --template`, `hint`, `route`, `ask`, `wake`, `host`) over `curator session` and `curator run`, and over today's `task-board session` commands until the node host lands; the login agent running `keeper up`; supervision through the extracted session host with capability tokens; routing with a generative fallback and routing hints; creating a process from a template; the main loop's own process on its own board | on one machine, a request typed in the terminal reaches the right one of three processes; a request for a missing process creates it from a template after the person confirms; after a reboot, `keeper up` brings back the main loop and every queen with their goals |
| KP2 | channels through waggle providers; the app's download link and the pairing QR code in the terminal; pairing with the companion app, with and without a screen, and approvals, grants and batched confirmations through it (on the app's own track) | a request written in a messenger is answered there, and an approval is signed on the phone; a headless node pairs without a screen |
| KP3 | the designer's generation and improvement proposals; the librarian on a schedule; skill discovery and installation from the Curator registry; the memory skill with its resolution ladder (fact cards, session, day, week and month summaries with identifiers, `memory expand`), files backend first; the designer's dream rehearsals and the nightly dream cycle (log distiller, replay, consolidation, dream memory) once curator-playbook dream mode ships | a request no template fits yields a validated new process; two near-duplicate playbooks yield a merge proposal that the person approves |
| KP4 | the sysadmin (backup to the person's own remotes, restore, the token inventory, the trusted-skills list, skill updates) and the watchdog (context, compaction, provider limits) | a node restored from its backup on a new machine runs the same processes with the same skill versions after the person confirms the secrets transfer on the app; a session about to compact writes its hand-off first |
| KP5 | several nodes: joined processes, a head main loop, several operators | one process is worked from two machines under leases, and a request typed on one machine reaches a process on another |

## Dependencies

- **Session host.**
  - Extract today's session daemon as a module first, unchanged (the operator's priority, 2026-09-24).
  - Then add what the main loop needs:
    - start a hosted session detached in any directory;
    - a session registry for `keeper session list` and the `keeper chat` picker; attach in control mode (`keeper chat`) or read-only (`keeper watch`);
    - park, stop and the emergency stop, with `task-board board snapshot` and the run directive `park` in the kernel;
    - its command line as the Curator provider `curator session …`, which keeper fronts;
    - per-session capability tokens that enforce who may act on which session;
    - deliver doorbells into sessions nobody is attached to;
    - telemetry for the watchdog: context usage, compaction signals, token events.
- **Curator.**
  - `curator run` as the one entry point (Decision 0021).
  - Environment profiles installed from a package directory, with machine overlays (Decision 0010): the `apiary` profile and the system roles' profiles are the install unit (specification §6.2, §13.3).
  - Source shorthands and default directories (curator-spec issue #92); `profile export` and `profile restore` of profile declarations (#93); the attestation fast path and a signed bootstrap (#94); command-only installs, `curator global add … --commands-only` (#95).
  - Provider installs: first-party providers beside `curator` and `curator provider install|check` for the rest, since a profile cannot carry `curator-run` (curator-spec issue #100, curator issue #96).
  - The operator-file commands `curator models` and `curator runtimes`, which hold the default harness as an `@orchestrator` binding.
  - Template packages instantiated without an upstream.
  - Commands built from source with the system's Go when it fits, never overriding it (curator issue #87); signed binaries later.
  - The registry of audited skills.
- **waggle.**
  - The local carrier, the doorbell and `task-board mail` (CM1), whose `approve` and `reject` back `keeper approve|reject`.
  - NATS and leases across machines (CM2).
  - Messenger providers.
  - Amendment A1 (board acts, hand-off results, delegation).
  - A companion-app signer provider.
- **curator-playbook.**
  - Processes, hand-off, delegation semantics, skill policy, template parameters (DRAFT v4.8, the core; v4.4 renamed its commands to the adopted command-line shape).
- **curator-trust.**
  - Who counts as a person, confirmations and their batching, capability grants, yolo, standing answers under review, the companion app and pairing, secrets, separation at the operating-system level (relux-works/curator-trust `spec/trust.md`), through the kernel's `task-board trust …`, which `keeper confirm|grant|revoke|device` front.
- **curator-inference-manager.**
  - The first-run local model, with download sizes the installer can state.
- **The operating system.**
  - A user-level login agent (launchd, systemd) and a keychain for tokens.
- **The companion app.**
  - Its own track, not started: one app over one secure-storage abstraction, on every platform, starting with one.

## Risks

- **The main loop becomes a bottleneck.**
  - Keep it thin. Once started, processes work without it, and they hand work to each other directly.
- **Misrouting.**
  - Every process applies its own rules, so a misrouted request cannot do more than that process allows.
  - Close candidates get a question.
  - Hints are visible and removable.
- **Idle cost.** An always-on harness session costs money.
  - A local model through a profile covers the front.
  - Our own loop comes later if needed.
- **Background orchestrators never hear their doorbell.** This happens if delivery keeps requiring an attached client, so the session host requirement comes first.
- **Authority creeps into the main loop.**
  - The invariants keep it out: no board writes and no spawns inside processes.
  - Registry changes need confirmation.
  - The authority table for sessions only narrows (specification §10.2).
- **The phone as the only approval device.**
  - Operating-system authentication on the machine stays a second path, and the app also runs on a computer.
  - Losing or replacing the phone is an open question.
- **A backup leaks secrets.** Secrets never enter git or an agent's environment; they move only between the node's keychain and the paired app, after the person confirms.
- **An update path becomes an injection path.** No node keeps an upstream to our repositories; our code enters only as audited, pinned packages from the Curator registry, and every change to a process still waits for confirmation.
- **Everything runs as one operating-system user before CM2.** Capability tokens and the operator's terminal are a guardrail, not a boundary; the emergency stop and confirmations need a person present. Protecting the root and the operator's files at the operating-system level is an open question.
