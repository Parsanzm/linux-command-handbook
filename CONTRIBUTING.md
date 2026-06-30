# Contributing to Linux Command Handbook

First off, thank you for considering contributing. This project only grows through community effort, and every page — whether a full command write-up or a one-line typo fix — genuinely helps.

---

## Ways to contribute

- **Write a new command page** from the "Coming soon" list in the [README](README.md)
- **Expand or improve an existing page** with more examples, missing edge cases, or clearer explanations
- **Fix mistakes** — typos, broken commands, outdated flags, incorrect output
- **Suggest a command** that isn't tracked yet by opening an issue
- **Report problems** you find while reading, even if you don't have time to fix them yourself

No contribution is too small. Fixing one typo is just as welcome as writing a full page.

---

## Before you start

1. Check the **Coming soon** list in the [README](README.md) and the open issues to avoid duplicate work.
2. If you're writing a new command page, open an issue (or comment on an existing one) saying which command you're taking, so two people don't write the same page at once.
3. For small fixes (typos, broken examples), you can skip straight to a pull request — no need to open an issue first.

---

## Adding a new command page

Each command lives in its own folder:

```
commands/<category>/<command-name>/
├── README.md
├── examples.md
├── edge-cases.md
└── interview-questions.md
```

Pick the category that fits best (`navigation`, `files`, `text-processing`, `search`, `networking`, `permissions`, `processes`, `disk`, `archives`, `system`, `user`). If you're unsure which category fits, just ask in your issue or PR — it's an easy thing to fix later.

### Use an existing page as your template

Don't start from a blank page. Read through one of the completed guides first to match the depth and tone expected:

- [`commands/text-processing/awk/`](commands/text-processing/awk/) — best example of a deep, language-like command
- [`commands/search/find/`](commands/search/find/) — best example of a command with many flags/options
- [`commands/user/useradd/`](commands/user/useradd/) — best example of a system administration command

### `README.md` — the complete reference

This is the main file. Structure that's worked well so far:

```markdown
# command — The Complete Reference

> **One-line summary of what it does**
> A short paragraph giving context — why it exists, what makes it useful.

---

## Table of Contents
(linked list of every section below)

## What is `command`?
What it does, a bit of history (who wrote it, when, why), and the core idea behind it.

## Where does `command` live?
Binary path, how to check the version/implementation, install instructions if it's not always preinstalled.

## How `command` works internally
The mental model — what actually happens when you run it.

## Syntax
The general command-line syntax pattern.

## Options / Flags
A table of flags with short descriptions, and the most important ones explained in more depth.

## Related Commands
Links to similar or complementary commands (including ones already documented here, if relevant).
```

Adjust section names to fit the specific command — `find` has "Tests" and "Actions" instead of a flat flag table, `awk` has "Patterns" and "Built-in Functions". Follow what makes sense for that command rather than forcing a rigid template.

### `examples.md`

Real, runnable, copy-pasteable examples — not abstract syntax. Group them by use case (e.g. "Basic usage", "Combining with pipes", "Real-world scenarios") and explain briefly *why* you'd use each one, not just *what* it does.

### `edge-cases.md`

The things that bite people: surprising defaults, platform differences (GNU vs BSD), symlink behavior, what happens with spaces in filenames, permission edge cases, etc. This is often the most valuable file for experienced users — don't skip it.

### `interview-questions.md`

Realistic questions someone might be asked about this command in a technical interview, each with a clear, complete answer. Mix conceptual questions ("What's the difference between X and Y?") with practical ones ("Write a command that does Z").

---

## Style guidelines

- Write in clear, direct English — short paragraphs, no filler.
- Use fenced code blocks with the `bash` language tag for commands and output.
- Prefer showing a command **and** its output over describing the output in prose.
- Link to related command pages within the handbook where it helps (e.g. `find` linking to `xargs` once that page exists).
- Keep formatting consistent with the existing pages — headers, tables, and code block style should look like they belong to the same book.

---

## Submitting your contribution

1. Fork the repository and create a branch (e.g. `docs/chmod` or `fix/grep-typo`).
2. Make your changes, following the structure and style above.
3. Open a pull request with a short description of what you added or fixed.
4. One command per pull request is preferred — it's easier to review and merge quickly.

Don't worry about getting everything perfect on the first try. Reviews are meant to help polish the page, not gatekeep it.

---

## Questions?

If anything here is unclear, or you're not sure where something belongs, just open an issue and ask. Happy to help get your first contribution in.
