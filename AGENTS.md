# AGENTS.md

## Introduction

This repository is a collection of study notes, including course notes, code analysis, topical studies, reference curation, and related content.

## Core Directories

- `notes/` - Each direct subdirectory is one note topic.
- `.agents/skills/` - Repository-specific AI agent skills. Read the relevant skill before non-trivial work.

## Agent Workflow

- Choose the applicable skill from `.agents/skills/` before changing notes, indexes, topic structure, or code-analysis materials.
- Use `.agents/skills/maintain-notebook-notes/SKILL.md` for general note maintenance, topic naming, links, and indexes.
- Use `.agents/skills/analyze-code-repository/SKILL.md` for `notes/code-ana_*` topics, source submodules, reference indexes, and code-analysis reports.
- Treat skills as task instructions, not as memory or task logs. Do not record one-off progress, short-lived task notes, transient failures, or conversation-derived memories in skills.
- Update skills only when the user explicitly requests a repository AI-rule change or when the current task is specifically to maintain those rules.

## Note Organization

- Use regular topic folder names under `notes/`, for example:
  - `course_${School}_${CourseName}` for course notes.
  - `code-ana_${RepoName}` for code repository analysis.
  - `study_${Subject}` for topical studies.
- Keep topic-level content inside its topic directory unless a cross-topic link is intentionally needed.

## Index Rules

- `notes/INDEX.md` indexes all topics by concise summaries. Prefer linking topic entries, not individual files inside each topic.
- `notes/${TopicName}/INDEX.md` indexes files inside that topic and gives the suggested reading path.
- Keep every `INDEX.md` optimized for quick search and on-demand loading.
- Before finishing changes, check and update every affected index, including `notes/INDEX.md` and topic-level indexes.

## Link Rules

- Use relative links within the same topic, for example `[intro](./intro.md)`.
- Use repo-root absolute links starting with `notes/` across topics, for example `[other](notes/xxx/other.md)`.
- In `notes/INDEX.md`, use relative links to topic directories.
