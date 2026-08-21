# iris-skills

[![skills.sh](https://skills.sh/b/freelabel/iris-skills)](https://skills.sh/freelabel/iris-skills)

Agent skills from the [IRIS](https://heyiris.io) platform team — the ones that are useful
outside our own codebase.

Each directory is a skill: a `SKILL.md` that teaches an agent how to do one job well.
They work with Claude Code, and with any agent runtime that reads the `skills` convention.

## Install

```bash
npx skills add freelabel/iris-skills
```

Or copy a single directory into your own `.claude/skills/`.

## What's here

| Skill | What it does |
|---|---|
| `agentic-loop` | Structure a build → judge → revise loop that stops on a real condition instead of running forever. |
| `architecture-review` | Run seven frameworks (SWOT, GAP, SEARCH, STRIDE, ATAM, C4, ADR) against a proposed change *before* any code is written. |
| `health-check` | A minimal, honest service health pass. |
| `playwright-tests` | Write, run, and debug Playwright end-to-end tests, including auth setup and the failure modes that waste the most time. |
| `stress-test` | Adversarial testing — unsolvable tasks, confidently-wrong output, same-model blind spots, and cost runaways. |

## A note on what is *not* here

We keep roughly fifty skills internally. Most are not here on purpose:

- anything touching client or patient data
- runbooks that only mean something inside our own repositories
- skills we vendored from other people — those belong to their authors

If a skill here still assumes too much about our setup, that is a bug. Open an issue.

## License

MIT
