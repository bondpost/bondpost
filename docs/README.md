# Documentation

These documents cover the reference implementation and the project around it. The protocol itself (the rules a second implementation must follow) is in the [spec repository](https://github.com/bondpost/spec).

| Document | Read it for |
| --- | --- |
| [Business requirements](business-requirements.md) | Why Bondpost exists, goals and non-goals, BR-01 to BR-15, the release roadmap, risks and open decisions |
| [User stories](user-stories.md) | 32 stories in six epics, with acceptance criteria that turn into tests, and the build order |
| [Architecture](architecture.md) | Components, library surface, CLI, bond states, verifier API, relay plugin, FR and NFR tables |
| [Test plan](test-plan.md) | The regtest walk-through, the required test kinds and the release gates |
| [Developer guide](developer-guide.md) | Reading path, milestones M0 to M10, the core logic sketch, common mistakes, grants |

## Where things go

| If you are changing… | Edit |
| --- | --- |
| The script, tweak, event tags, verification steps, weight formula | The spec repository first, then this code |
| CLI flags, HTTP API, storage, plugin config | [Architecture](architecture.md) |
| Scope, priorities, releases | [Business requirements](business-requirements.md) and [user stories](user-stories.md) |
| A design choice with real alternatives | A decision file in the spec repository's [`decisions/`](https://github.com/bondpost/spec/tree/main/decisions) |

Requirement IDs trace from one document to the next: BR → FR/NFR → user story → milestone.
