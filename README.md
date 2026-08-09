# `nrminor/skills`

A small collection of reusable guidance I use with my coding agents. Each skill is a self-contained `SKILL.md` in the portable [Agent Skills](https://agentskills.io/specification) format.

## Skills

| Skill                                                                                    | Use it when                                                                  |
| ---------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------- |
| [`allocations`](skills/allocations/SKILL.md)                                             | Auditing or reducing allocations in Rust, TypeScript, JavaScript, or Python. |
| [`back-of-envelope`](skills/back-of-envelope/SKILL.md)                                   | Running a quick disposable experiment to check an assumption or API shape.   |
| [`code-sketches`](skills/code-sketches/SKILL.md)                                         | Comparing API, refactor, or architecture designs with concrete sketches.     |
| [`codebase-searcher`](skills/codebase-searcher/SKILL.md)                                 | Searching a codebase efficiently with structural and textual tools.          |
| [`conceptual-pca`](skills/conceptual-pca/SKILL.md)                                       | Sharpening muddy terminology and concepts in a codebase.                     |
| [`effect-ts`](skills/effect-ts/SKILL.md)                                                 | Writing or reviewing TypeScript built with Effect V4.                        |
| [`jj`](skills/jj/SKILL.md)                                                               | Working with Jujutsu using a graph-aware, describe-first workflow.           |
| [`jj-preemptive-conflict-resolution`](skills/jj-preemptive-conflict-resolution/SKILL.md) | Resolving or avoiding difficult conflicts in a Jujutsu commit graph.         |
| [`local-ci`](skills/local-ci/SKILL.md)                                                   | Developing and testing GitHub Actions workflows locally with `wrkflw`.       |
| [`logging`](skills/logging/SKILL.md)                                                     | Adding useful structured logging based on wide events.                       |
| [`performance-design`](skills/performance-design/SKILL.md)                               | Preserving a high performance ceiling during architecture and planning.      |
| [`self-doubt`](skills/self-doubt/SKILL.md)                                               | Stress-testing a proposed solution before committing to it.                  |

## Installing

Use the [`skills`](https://github.com/vercel-labs/skills) installer and choose which skills and coding agents to configure:

```bash
pnpm dlx skills@latest add nrminor/skills
```

To inspect the available skills without installing them:

```bash
pnpm dlx skills@latest add nrminor/skills --list
```

## OpenCode commands

The repository also includes six optional commands under [`.opencode/commands`](.opencode/commands):

| Command                   | Purpose                                                   |
| ------------------------- | --------------------------------------------------------- |
| `complete-next-task`      | Complete the next unfinished task in a PRD.               |
| `describe-commit`         | Describe the current Jujutsu commit from its intent.      |
| `query`                   | Translate a data question to SQL and run it with DuckDB.  |
| `run-multiphase-review`   | Coordinate parallel specialist reviews.                   |
| `tidy-commit-graph`       | Find and clean up untidy intermediate Jujutsu commits.    |
| `update-working-document` | Bring a working document in line with the implementation. |

Clone the repository and copy or link the desired files into `~/.config/opencode/commands/` to install them.
