# Repository Guidelines

## Project Structure & Module Organization
This repository is a course-material workspace, not a software package. The root currently contains:

- `HNU_计算机体系结构_完整学习手册.md`: the primary editable study guide and main source of truth.
- `计算机体系结构量化研究方法第6版 ... .pdf`: reference textbook material; treat it as read-only source content.

Keep new contributor-authored material in Markdown when possible. If additional notes, figures, or exercise files are added later, place them in clearly named top-level folders such as `notes/`, `figures/`, or `exercises/` rather than mixing drafts with reference files.

## Build, Test, and Development Commands
No build system or automated test suite is configured in this repository today. Use lightweight local checks:

- `rg --files`: list tracked workspace files quickly.
- `rg -n "^#"` `HNU_计算机体系结构_完整学习手册.md`: verify heading structure and section depth.
- `Get-Content -Path .\HNU_计算机体系结构_完整学习手册.md -TotalCount 40`: inspect the opening section after edits.

Preview Markdown in your editor before finalizing changes, and confirm headings, lists, and formulas render correctly.

## Coding Style & Naming Conventions
Use Markdown as the default authoring format. Prefer short sections, consistent heading depth, and concise instructional prose. Keep list indentation consistent with standard Markdown conventions and avoid mixing bullet styles within the same section.

Preserve the repository’s descriptive Chinese file naming where it improves clarity. For newly added files, use explicit names that reflect course scope, for example `作业3_缓存题解析.md`.

## Testing Guidelines
There is no formal coverage target. Validation is editorial:

- check heading hierarchy;
- verify formulas, tables, and lists render correctly;
- confirm references to chapters, assignments, and labs match the source material.

When making substantial content changes, review the surrounding section to keep terminology and difficulty level consistent.

## Commit & Pull Request Guidelines
No Git history is available in this workspace, so there is no repository-specific commit convention to infer. If version control is introduced, use short imperative commit messages with a scope prefix when helpful, such as `docs: refine Tomasulo experiment notes`.

Pull requests should summarize the edited sections, note any source used for factual corrections, and include screenshots only when formatting changes materially affect Markdown rendering.

## Source Handling
Do not overwrite the PDF reference file. Make substantive edits in Markdown, and document any interpretation or restructuring of textbook content in the relevant section rather than changing source wording silently.
