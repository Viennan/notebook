# AGENTS.md

## Introduction

This project is a collection of study notes, including but not limited to course notes, code analysis, topical studies, and other related content.

## Core Project Directories

- notes - Each subdirectory under the `notes` directory represents a specific `topic`.
- docs - Experience for an ai agent to maintaining this project.

## Engineering Rules

- Use regular patterns to name folders under `notes`. eg:
  - `course_${School}_${CourseName}` represents course notes.
  - `code-ana_${ReposName}` represents code analysis.
- Index maintaining rules:
  - `notes/INDEX.md` - Index all `topic` by means of summaries. It is recommended to avoid linking files inside a topic.
  - `notes/${TopicName}/INDEX.md` - The index for files of the specific `topic`.
  - All `INDEX.md` should be optimized for quick searching and on-demand loading.
- File link rules
  - Use relative link within the same `topic`, eg: "[xxx](./intro.md)".
  - Use absolute link (starting with "notes") across different `topic`, eg: "[xxx](notes/xxx/xxx.md)"
  - `notes/INDEX.md` should use relative link.
- Experience docs:
  - The `docs` directory is reserved for best practices related to this project's maintenance.
