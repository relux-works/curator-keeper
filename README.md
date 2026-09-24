# curator-keeper

The main loop of an agent system (working name: Keeper). On one machine it keeps the registry of the processes that machine runs — a family calendar, a café, a hobby project. It routes each request a person makes to the right process, starts and supervises each process's root orchestrator, and talks with the person.

## Status

**Draft design, v0.** Nothing is implemented. The specification is a draft (milestone KP0); user-facing names are pending the operator's decision (spec §2.1).

## Layout

| Path | Content |
| --- | --- |
| [`spec/keeper.md`](spec/keeper.md) | the draft specification |
| [`spec/track.md`](spec/track.md) | milestones KP0–KP4, dependencies, risks |
| [`diagrams/`](diagrams/) | diagrams as code (PlantUML) and their renders |

## How it fits

- **Processes** are configured by curator-playbook (`Playbook.json` per project). The main loop starts exactly one root orchestrator per process and never does process work itself.
- **Sessions** are started, kept alive and woken by the session-host module, which is being extracted from task-board's session daemon.
- **Messages** between the main loop and orchestrators are waggle messages: a doorbell, then a pull.
- **People** talk through channels and approve through a companion app or the machine's user authentication.

The ecosystem map, decisions and open questions live in the private wiki (`relux-works/wiki`).
