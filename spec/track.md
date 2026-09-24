# Track: main loop

Status: **DRAFT v0**, 2026-09-24. Specification: [`keeper.md`](keeper.md). Roadmap: `wiki/roadmap/ecosystem-roadmap.md`, line KP.

## Why

The process configuration (curator-playbook) lets one project describe how it works. One person runs several such processes on one machine — a family calendar, the smart home, a hobby project, a café — and needs one place that knows them all. That place takes a request in plain words, sends it to the right process, keeps that process's orchestrator running, and brings back results and questions. Without it, the person has to open the right orchestrator by hand and remember how everything is organized.

## Milestones

| # | Delivers | Done when |
| --- | --- | --- |
| KP0 | draft specification, diagrams, a refuter run | merged as a draft; every refuter finding resolved or listed as open |
| KP1 | one node on existing harnesses: the keeper skill and the `keeper` command; the process registry; `keeper start` and `keeper attach` through the extracted session host; starting and waking root orchestrators; routing with a generative fallback and routing hints; creating a process from a template; the main loop's own process on its own board | on one machine, a request typed in the terminal reaches the right one of three processes, and a request for a missing process creates it from a template after the person confirms |
| KP2 | channels through waggle providers; pairing with the companion app and approvals through it (on the app's own track) | a request written in a messenger is answered there, and an approval is signed on the phone |
| KP3 | the designer's generation and improvement proposals; the librarian on a schedule; skill discovery and installation from the Curator registry | a request no template fits yields a validated new process; two near-duplicate playbooks yield a merge proposal that the person approves |
| KP4 | several nodes: joined processes, a head main loop, several operators | one process is worked from two machines under leases, and a request typed on one machine reaches a process on another |

## Dependencies

- **Session host.**
  - Extract today's session daemon as a module first, unchanged (the operator's priority, 2026-09-24).
  - Then add what the main loop needs:
    - start a hosted session detached in any directory;
    - attach to any session, with viewers;
    - park and stop;
    - deliver doorbells into sessions nobody is attached to.
- **Curator.**
  - `curator run` as the one entry point (Decision 0021).
  - Environment profiles (Decision 0010), including the main loop's profile.
  - On-demand skills and the skills launch channel (curator-spec issue #91).
  - Skill commands built from source.
  - The registry of audited skills.
- **waggle.**
  - The local carrier, the doorbell and `task-board mail` (CM1).
  - NATS and leases across machines (CM2).
  - Messenger providers.
  - Amendment A1 (board acts, hand-off results, delegation).
  - A companion-app signer provider.
- **curator-playbook.**
  - Processes, confirmation of authority-bearing sections, hand-off, delegation, standing answers.
- **curator-inference-manager.**
  - The first-run local model.
- **The companion app.**
  - Its own track, not started.

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
- **The phone as the only approval device.**
  - Operating-system authentication on the machine stays a second path.
  - Losing or replacing the phone is an open question.
