# Diagrams

Diagrams as code for the main-loop specification ([`../spec/keeper.md`](../spec/keeper.md)).

## Structure

```
diagrams/
├── README.md                    this file: tools, commands, conventions
├── plantuml/
│   ├── _includes/theme.puml     shared look (the same as the wiki's), included by every diagram
│   ├── activity/                flows of steps
│   ├── deployment/              structure: what runs where
│   └── sequence/                runtime interactions
└── artefacts/plantuml/          rendered SVG and PNG, committed
```

## Diagrams

| Diagram | Kind | Source | Renders |
| --- | --- | --- | --- |
| One node: the main loop and its processes | PlantUML deployment | [one-node-system.puml](plantuml/deployment/one-node-system.puml) | [SVG](artefacts/plantuml/one-node-system.svg), [PNG](artefacts/plantuml/one-node-system.png) |
| Several nodes | PlantUML deployment | [several-nodes.puml](plantuml/deployment/several-nodes.puml) | [SVG](artefacts/plantuml/several-nodes.svg), [PNG](artefacts/plantuml/several-nodes.png) |
| Installation and first run | PlantUML activity | [installation-and-first-run.puml](plantuml/activity/installation-and-first-run.puml) | [SVG](artefacts/plantuml/installation-and-first-run.svg), [PNG](artefacts/plantuml/installation-and-first-run.png) |
| Routing a request | PlantUML sequence | [request-routing.puml](plantuml/sequence/request-routing.puml) | [SVG](artefacts/plantuml/request-routing.svg), [PNG](artefacts/plantuml/request-routing.png) |
| Reboot: `keeper up` restores the node | PlantUML sequence | [reboot-restore.puml](plantuml/sequence/reboot-restore.puml) | [SVG](artefacts/plantuml/reboot-restore.svg), [PNG](artefacts/plantuml/reboot-restore.png) |

## Tools

| Tool | Used for | Install |
| --- | --- | --- |
| PlantUML (the renders above: 1.2026.6, except `installation-and-first-run` and `reboot-restore`: 1.2026.8) | turns each `.puml` source into SVG and PNG | `brew install plantuml`, or the JAR: `curl -L -o plantuml.jar https://github.com/plantuml/plantuml/releases/latest/download/plantuml.jar` |
| Graphviz `dot` (the renders above: 14.0.4) | the layout engine PlantUML uses for deployment diagrams | `brew install graphviz` |

`plantuml -testdot` checks that PlantUML finds Graphviz.

## Render

From the repository root:

```bash
plantuml -tsvg -o "$PWD/diagrams/artefacts/plantuml" diagrams/plantuml/activity/*.puml diagrams/plantuml/deployment/*.puml diagrams/plantuml/sequence/*.puml
plantuml -tpng -o "$PWD/diagrams/artefacts/plantuml" diagrams/plantuml/activity/*.puml diagrams/plantuml/deployment/*.puml diagrams/plantuml/sequence/*.puml
```

The output directory is absolute because PlantUML resolves a relative `-o` against each source file's folder.

## Conventions

- One purpose per diagram, with a title and a caption naming the specification sections it follows.
- Technical terms in labels, as in the specification's prose; beekeeping names appear only where the specification uses them too (command names, directories and profiles).
- Colour says what a thing is:
  - blue: node services (session host, commands);
  - purple: agents (the main loop session, orchestrators, runs);
  - amber: committed files and boards;
  - grey: the operator's private files;
  - green: people and the companion app;
  - teal: the waggle carrier.
- A change to a source is committed together with its re-rendered SVG and PNG.
