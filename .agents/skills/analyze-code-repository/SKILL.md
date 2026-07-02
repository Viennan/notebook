---
name: analyze-code-repository
description: Create and maintain code repository analysis topics under notes/code-ana_${RepoName}/, including source submodules, ref/INDEX.md, VS Code watcher exclusions, source-grounded reports, and topic indexes. Use when analyzing an external code repository, initializing or updating a code-ana topic, recording source commit metadata, curating authoritative references, writing architecture or mechanism reports, or checking code-analysis topic hygiene.
---

# Analyze Code Repository

## Overview

Code analysis notes should answer how a repository's key capability is organized, executed, and constrained. Reports should build a mainline understanding first, then point to source details as evidence.

Also apply `../maintain-notebook-notes/SKILL.md` for general topic, index, and link rules.

## Topic Layout

Use this layout for new code-analysis topics:

```text
notes/code-ana_${RepoName}/
├── INDEX.md
├── ref/
│   └── INDEX.md
├── ${repo}/
├── ${repo}-core-report.md
└── ${repo}-${topic}.md
```

- `INDEX.md` summarizes scope, reading order, source commit, and report entry points.
- `ref/INDEX.md` indexes official or authoritative references.
- `${repo}/` is the analyzed upstream repository, normally added as a git submodule.
- Report filenames should be short, stable, and mechanism-oriented.

## Submodule Workflow

Prefer storing the analyzed repository as a submodule inside its topic directory. Ask the user to run submodule initialization commands locally, then continue after confirmation.

```bash
mkdir -p notes/code-ana_${RepoName}
git submodule add ${RepositoryURL} notes/code-ana_${RepoName}/${repo}
```

If `.gitmodules` already exists or the submodule path is already configured, ask the user to run:

```bash
git submodule update --init --recursive notes/code-ana_${RepoName}/${repo}
```

After initialization, record:

- Upstream repository URL.
- Current commit.
- Branch or tag used for analysis.
- Whether the repository itself has submodules.
- The analysis scope and any intentionally excluded areas.

Useful checks:

```bash
git -C notes/code-ana_${RepoName}/${repo} status --short
git -C notes/code-ana_${RepoName}/${repo} rev-parse HEAD
git -C notes/code-ana_${RepoName}/${repo} submodule status --recursive
```

## Watcher Exclusions

When adding, moving, or removing a source submodule, update `.vscode/settings.json` `files.watcherExclude`.

- Exclude only the source submodule path, not the whole topic directory.
- Preserve existing user settings.
- Match the repository's current glob style when possible, for example `**/code-ana_${RepoName}/${repo}/**`.

## Reference Index

Maintain `ref/INDEX.md` for official or authoritative sources:

- Official docs, upstream repository links, releases, specs, papers, or first-party announcements.
- Local source docs such as README, CONTRIBUTING, SECURITY, `docs/`, or website docs.
- A short purpose note for each reference: product context, API reference, design doc, security boundary, and so on.
- Last checked date, version, applicability, or staleness risk when useful.

Do not copy large source text into the notebook. Link to it and summarize why it matters.

## Source Hygiene

Default to read-only treatment of the submodule source tree.

- Do not modify source, lockfiles, configs, generated snapshots, or formatting in the analyzed repository.
- Allowed exception: dependency or tool output under paths already ignored by the source repository, such as `node_modules/`, `.turbo/`, `target/`, `.venv/`, or `.gradle/`.
- Before running commands that may write files, confirm the target path is ignored.
- If non-ignored changes appear, pause and inspect them. Do not revert user changes.

```bash
git -C notes/code-ana_${RepoName}/${repo} check-ignore -v node_modules/
git -C notes/code-ana_${RepoName}/${repo} status --short
```

## Dependency Initialization

Initialize dependencies only to improve code reading, type navigation, or local search. Prefer frozen, no-script, no-lockfile-change commands.

First identify the stack:

```bash
rg --files notes/code-ana_${RepoName}/${repo} -g 'package.json' -g 'bun.lock' -g 'pnpm-lock.yaml' -g 'package-lock.json' -g 'yarn.lock' -g 'Cargo.toml' -g 'go.mod' -g 'pyproject.toml'
```

Preferred commands:

| Stack | Command |
| --- | --- |
| Bun / TypeScript | `bun install --frozen-lockfile --ignore-scripts` |
| pnpm / TypeScript | `pnpm install --frozen-lockfile --ignore-scripts` |
| npm / TypeScript | `npm ci --ignore-scripts` |
| Yarn classic | `yarn install --frozen-lockfile --ignore-scripts` |
| Rust | `cargo fetch --locked` |
| Go | `go mod download` |
| Python | Create `.venv` only when `.venv/` is ignored, then install from lock/requirements |
| Maven | `mvn -q -DskipTests dependency:go-offline` |
| Gradle | `./gradlew --no-daemon dependencies` after confirming `.gradle/` is ignored |

If install scripts or generators are required for type files or native bindings, explain the reason before running a broader command.

## Analysis Flow

1. Define the analysis scope and explicitly exclude unrelated UI, enterprise, configuration, or migration edge cases when appropriate.
2. Find the main execution path first: entry points, core loop, domain model, state transitions, and service boundaries.
3. Deepen by question, not by directory. Each report should answer a specific product or architecture question.
4. Read source, tests, prompts, tool descriptions, and docs together. Source at the recorded commit is the authority; docs provide intent and user-visible framing.
5. Separate facts from interpretation. Architecture claims should point back to concrete source behavior.

## Report Style

- Start with the core conclusion and a source map.
- Prefer why / what / how / evidence: purpose and pressure, capability boundaries, core objects and state, runtime path and lifecycle, then source or test evidence.
- Avoid per-file summaries. File paths, classes, and functions should support the conceptual model.
- Use diagrams, sequence flows, state tables, or comparison tables when they clarify lifecycle or data flow.
- Keep code snippets short; link to source for longer evidence.
- For migrating or dual-track architectures, mark old path, target architecture, and the boundary between them.
- End with design philosophy, limitations, and risks when useful.

## Final Checklist

- The topic `INDEX.md` and `notes/INDEX.md` are updated if scope, files, or summaries changed.
- `ref/INDEX.md` exists and links relevant authoritative references.
- Source commit and analysis scope are recorded.
- `.vscode/settings.json` excludes the source submodule path only.
- The submodule has no non-ignored changes.
- The report explains the mainline behavior before implementation details.
