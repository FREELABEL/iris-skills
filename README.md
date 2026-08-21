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

## A note on what is *not* here

We keep roughly fifty skills internally. Most are not here on purpose:

- anything touching client or patient data
- runbooks that only mean something inside our own repositories
- skills we vendored from other people — those belong to their authors
- anything that assumes our infrastructure, even after the obvious references are stripped

Two skills were published here briefly and then withdrawn, for the last two reasons.
A Playwright guide still assumed our ports, our spec filenames and our directory layout
once the surface details were scrubbed — it was a runbook wearing a skill's clothes. An
adversarial testing skill generated injection and auth-bypass batteries and, by its own
description, ran them against production; that is a reasonable internal tool and not a
reasonable thing to hand a stranger. Neither was unsafe to read. Both were misleading to
install, which is the bar that matters here.

If a skill here still assumes too much about our setup, that is a bug. Open an issue.

## License

MIT
