# AGENTS.md

## Repo Purpose

A collection of Android development reference guides. Each topic has two files:
- `<Topic>.md` — full in-depth guide (root directory)
- `Revisions/<Topic>_Revision.md` — condensed last-minute revision summary of the same topic

## File Inventory

| Guide | Revision |
|---|---|
| `Accessibility Guide.md` | `Revisions/Accessibility Guide_Revision.md` |
| `Android Animations Guide.md` | `Revisions/Android Animations Guide_Revision.md` |
| `Android Design Patterns.md` | `Revisions/Android Design Patterns_Revision.md` |
| `Android Rendering Pipelines.md` | `Revisions/Android Rendering Pipelines_Revision.md` |
| `Android Service Guide.md` | `Revisions/Android Service Guide_Revision.md` |
| `Multithreading in Android with Java.md` | `Revisions/Multithreading in Android with Java_Revision.md` |
| `Periodic Location Tracking.md` | `Revisions/Periodic Location Tracking_Revision.md` |
| `UI Performance & Memory Leaks.md` | `Revisions/UI Performance & Memory Leaks_Revision.md` |

## Conventions

- Every guide in the root must have a matching `_Revision.md` in `Revisions/`. Keep them in sync when editing.
- Revision files are condensed summaries — bullet points, tables, key facts only. Do not pad them with prose from the full guide.
- Full guides use numbered Table of Contents sections. Maintain that structure when adding content.
- No build system, no tests, no code — this is a pure documentation repo.

## Editing Rules

- When adding a new topic, create both the full guide and the revision file.
- Revision filenames follow the pattern `<Exact Guide Name>_Revision.md` (spaces preserved, not replaced with underscores).
- The `.idea/` directory is an Android Studio project config — ignore it unless opening in Android Studio.
