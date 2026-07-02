---
name: maintain-notebook-notes
description: Maintain this notebook repository's note topics, topic indexes, root notes index, naming conventions, and link conventions. Use when creating, moving, renaming, deleting, or updating files under notes/, editing any INDEX.md, adding a new topic, changing note navigation, curating references, or reorganizing repository note content.
---

# Maintain Notebook Notes

## Overview

Keep the repository easy for future agents and humans to search, load, and extend. Prefer small, navigable topic updates over broad rewrites.

## Workflow

1. Identify the work type: course notes, code analysis, topical study, reference curation, index maintenance, or another topic pattern.
2. Find the affected topic directory under `notes/`.
3. Apply the naming, index, and link rules below.
4. Check every affected `INDEX.md` before finishing.

For `notes/code-ana_*` work, also use `../analyze-code-repository/SKILL.md`.

## Topic Naming

- Use `course_${School}_${CourseName}` for course notes.
- Use `code-ana_${RepoName}` for code repository analysis.
- Use `study_${Subject}` for topical studies.
- Keep names stable once links exist. If a topic must be renamed, update all affected links and indexes in the same change.

## Index Rules

- `notes/INDEX.md` lists all topics with concise summaries. Link topic entries, not every file inside a topic.
- `notes/${TopicName}/INDEX.md` maps that topic's files, reading order, and current scope.
- Optimize indexes for quick search and on-demand loading: short summaries, stable labels, and links to entry points.
- Update parent and topic indexes when adding, deleting, moving, or renaming content.
- Do not let index pages become long-form notes; move detailed content into topic files.

## Link Rules

- Use relative links inside one topic, for example `[intro](./intro.md)`.
- Use repo-root links beginning with `notes/` across topics, for example `[model spec](notes/study_openai/model-spec.md)`.
- In `notes/INDEX.md`, use relative links to topic directories, for example `[study_openai](./study_openai/)`.
- Prefer links to durable entry points such as `INDEX.md` or a named report instead of conversational scratch files.

## Reference Curation

- Prefer official, primary, or otherwise authoritative sources.
- Store URLs and a short purpose note instead of copying large external text into the notebook.
- Record version, date checked, or applicability only when it helps future readers judge staleness.
- Keep reference indexes near the topic that uses them.

## Scope Rules

- Do not refactor unrelated historical topics while making a focused update.
- Do not use `.agents/skills` as memory files or task progress logs.
- Remove obsolete navigation paths when moving content so future agents do not follow stale instructions.
