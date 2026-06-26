# AGENTS.md

## Introduction

This project is a collection of study notes, including but not limited to course notes, code analysis, topical studies, and other related content.

## Core Project Directories

- notes - Each subdirectory under the `notes` directory represents a specific `topic`.
- profile - Experience profile for AI agents maintaining this project.

## Engineering Rules

- Use regular patterns to name folders under `notes`. eg:
  - `course_${School}_${CourseName}` represents course notes.
  - `code-ana_${ReposName}` represents code analysis.
- Index maintaining rules:
  - `notes/INDEX.md` - Index all `topic` by means of summaries. It is recommended to avoid linking files inside a topic.
  - `notes/${TopicName}/INDEX.md` - The index for files of the specific `topic`.
  - All `INDEX.md` should be optimized for quick searching and on-demand loading.
  - Before committing, check and update every potentially affected `INDEX.md`, including parent-level and topic-level indexes.
- File link rules
  - Use relative link within the same `topic`, eg: "[xxx](./intro.md)".
  - Use absolute link (starting with "notes") across different `topic`, eg: "[xxx](notes/xxx/xxx.md)"
  - `notes/INDEX.md` should use relative link.
- Agent experience profile:
  - `profile/EXP.md` is the general entry point for durable maintenance experience, not a code-analysis-only guide.
  - Before non-trivial maintenance work, identify the work type first: course notes, code analysis, topical study, index maintenance, reference curation, or another topic pattern.
  - Read `profile/EXP.md` as a discovery map, then follow only the same-directory links that match the current work.
  - Treat linked profile docs such as `profile/code-analysis-best-practices.md` as on-demand topic guidance; add new linked docs when another topic type develops stable reusable practice.
  - After finishing work, consider whether a reusable lesson should update `profile/EXP.md` or a linked profile doc.
  - Keep detailed experience out of `AGENTS.md`; write it in `profile/EXP.md` or linked files under `profile/`.
  - Keep `profile/EXP.md` within 3000 Chinese characters by linking to focused same-directory documents for on-demand loading.
  - Update the profile only for stable lessons that reduce future user correction; do not record one-off task progress, temporary TODOs, or transient environment failures.
