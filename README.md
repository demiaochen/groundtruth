# groundtruth

A focused set of [Agent Skills](https://agentskills.io/specification) for verifying an AI coding
agent's UI and behavioral work.

A passing test suite and a clean build are the floor, not proof. The defects users actually hit
live in the UI, interaction, and OS glue that unit tests miss, and an agent reviewing its own
screenshots is fooled in predictable ways. These skills make it verify by running and observing
the real thing.

Each skill is a portable `SKILL.md` (YAML frontmatter plus Markdown) that loads into Claude Code,
Cursor, Codex, or any agent supporting the [open standard](https://agentskills.io/specification).

## Skills

| Skill | Summary |
| ----- | ------- |
| [visual-self-verification](./skills/visual-self-verification) | The four ways an LLM misreads its own screenshots (resting frames hide broken animations, downscaled renders invert contrast, interaction states never fire or get occluded, stale builds show old output), and the countermeasure for each. |
| [adversarial-qa-loop](./skills/adversarial-qa-loop) | Drive a project to zero defects with proof. Parallel finder lanes (static, contract-oracle, flow/journey, differential), a default-reject verifier with three verdicts, behavioral drives of the running software, sequential fixes with red-then-green tests, looped until a full sweep is clean. |
| [drive-the-fix](./skills/drive-the-fix) | Take one bug from report to a verified, landed fix. Reproduce the real trigger, root-cause, isolate, change it, then prove it by running the real artifact, with review ceremony scaled to blast radius. |

## Example

`visual-self-verification`, trap #1: a reveal animation looks correct in every screenshot because
a screenshot only captures the settled state. Record the transition and the panel slides over its
neighbor for ~200ms before snapping into place.

```
at rest (screenshot)        mid-transition (recording, t≈0.2s)
┌───────────────┐           ┌───────────────┐
│   panel       │  looks    │ panel ░░░░░    │  slides over its
│               │  correct  │   neighbor     │  neighbor, unclipped
└───────────────┘           └───────────────┘
```

Full walkthrough: [skills/visual-self-verification/README.md](./skills/visual-self-verification/README.md).

## Install

### npx skills

[`skills`](https://github.com/vercel-labs/skills) installs from this repo into 60+ agents
(Claude Code, Cursor, Codex, and others), detecting the ones you have.

```bash
npx skills add demiaochen/groundtruth                 # interactive: pick skills and agents
npx skills add demiaochen/groundtruth --list          # list skills in the repo
npx skills add demiaochen/groundtruth --skill visual-self-verification -a claude-code -a cursor
npx skills add demiaochen/groundtruth -g              # user-level (all projects)
```

Installs to `.agents/skills/` and symlinks into each agent's skills directory. Manage with
`npx skills list` and `npx skills update`.

### Claude Code plugin

```text
/plugin marketplace add demiaochen/groundtruth
/plugin install groundtruth@groundtruth
```

Skills load namespaced, for example `/groundtruth:visual-self-verification`. Update with
`/plugin marketplace update groundtruth`.

### Manual

```bash
git clone https://github.com/demiaochen/groundtruth.git
cp -R groundtruth/skills/visual-self-verification ~/.claude/skills/   # or .cursor/skills/, etc.
```

## Structure

```text
groundtruth/
├── skills/
│   └── <skill>/
│       ├── SKILL.md   # agent-facing instruction (agentskills.io standard)
│       └── README.md  # human-facing notes
└── .claude-plugin/    # Claude Code marketplace and plugin manifests
```

## License

[MIT](./LICENSE) © Demiao Chen
