# curator-keeper

The main loop of an agent system: the keeper of an apiary. On one machine it keeps the registry of the processes that machine runs — a family calendar, a café, a hobby project. It routes each request a person makes to the right process, starts and supervises each process's root orchestrator, and talks with the person.

## Status

**Draft design, v0.5.1.** Nothing is implemented. The specification is a draft (milestone KP0). The beekeeping names were approved on 2026-09-25 (spec, Names): apiary, yard, keeper, hive, queen, comb, bee, wild bees, nuc. v0.3 adds the memory resolution ladder, the designer's dream rehearsals and the command-only install of `keeper`, and records role profiles as decided. v0.5 applies the command-line shape adopted on 2026-09-27: beekeeping words only in `keeper`'s nouns and the apiary's directories, the full `keeper` command map over `curator session`, `curator run` and `task-board trust`, and the default harness as an operator binding.

## Layout

| Path | Content |
| --- | --- |
| [`spec/keeper.md`](spec/keeper.md) | the draft specification |
| [`spec/track.md`](spec/track.md) | milestones KP0–KP5, dependencies, risks |
| [`diagrams/`](diagrams/) | diagrams as code (PlantUML) and their renders |

## How it fits

- **Processes** are configured by curator-playbook (`Playbook.json` per project). The main loop starts exactly one root orchestrator per process and never does process work itself.
- **Sessions** are started, kept alive and woken by the session-host module, which is being extracted from task-board's session daemon; its command line is `curator session …`, and `keeper` fronts it.
- **Messages** between the main loop and orchestrators are waggle messages: a doorbell, then a pull.
- **People** talk through channels and approve through a companion app or the machine's user authentication.

The ecosystem map, decisions and open questions live in the private wiki (`relux-works/wiki`).
