# 🐧 Linux Command Handbook

> A practical, in-depth Linux command reference — built one command at a time, with real-world examples, edge cases, and interview-ready explanations.

![License](https://img.shields.io/badge/license-MIT-blue.svg)
![Status](https://img.shields.io/badge/status-actively%20maintained-success.svg)
![PRs](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)
![Progress](https://img.shields.io/badge/commands-12%20%2F%2052%20completed-yellow.svg)

This isn't a copy-paste cheat sheet. Every finished page covers what a command actually *is*, how it works internally, its full syntax, dozens of real examples, the edge cases that trip people up, and the kind of questions you'd get asked about it in an interview.

---

## 🚧 Project Status — This is a living project

**This handbook is continuously growing.** New command pages are added regularly, and every existing page can be expanded or corrected over time — nothing here is ever "final." If a command you're looking for isn't documented yet, it's most likely already scaffolded and waiting to be written next.

⭐ **Star or watch this repo** to follow new additions, and feel free to suggest or request a command via an issue.

**Current progress: 12 of 52 command pages fully completed (~23%).**

| Category | Progress | Status |
|---|---|---|
| 🔎 Search | 2 / 2 | ✅ Complete |
| 👤 User Management | 3 / 3 | ✅ Complete |
| 📝 Text Processing | 3 / 9 | 🚧 In progress |
| 🧭 Navigation | 2 / 6 | 🚧 In progress |
| 💽 Disk | 1 / 4 | 🚧 In progress |
| 📦 Archives | 1 / 2 | 🚧 In progress |
| 📁 Files | 0 / 6 | ⏳ Planned |
| 🌐 Networking | 0 / 6 | ⏳ Planned |
| 🔐 Permissions | 0 / 3 | ⏳ Planned |
| ⚙️ Processes | 0 / 5 | ⏳ Planned |
| 🛠️ System | 0 / 6 | ⏳ Planned |

---

## 📖 What's inside every command page

Each command lives in its own folder under `commands/<category>/<command>/` and follows the same four-file format:

| File | Purpose |
|---|---|
| `README.md` | The complete reference — what it is, how it works, full syntax, options, related commands |
| `examples.md` | Real-world, copy-pasteable usage examples |
| `edge-cases.md` | Gotchas, quirks, and situations that don't behave the way you'd expect |
| `interview-questions.md` | Common questions asked about this command, with answers |

---

## ✅ Completed guides — browse now

| Command | Category | Description |
|---|---|---|
| [`cd`](commands/navigation/cd/README.md) | Navigation | Change the current directory |
| [`ls`](commands/navigation/ls/README.md) | Navigation | List directory contents |
| [`grep`](commands/search/grep/README.md) | Search | Search text using patterns |
| [`find`](commands/search/find/README.md) | Search | Search for files in a directory hierarchy |
| [`awk`](commands/text-processing/awk/README.md) | Text Processing | Pattern scanning and text processing language |
| [`cat`](commands/text-processing/cat/README.md) | Text Processing | Concatenate and display file contents |
| [`less` / `more`](commands/text-processing/less-more/README.md) | Text Processing | Page through file contents |
| [`mount`](commands/disk/mount/README.md) | Disk | Mount filesystems |
| [`tar`](commands/archives/tar/README.md) | Archives | Archive and compress files |
| [`passwd`](commands/user/passwd/README.md) | User | Change a user's password |
| [`useradd`](commands/user/useradd/README.md) | User | Create a new user account |
| [`who` / `w`](commands/user/who-w/README.md) | User | Show who is logged in |

---

## ⏳ Coming soon

These commands are already scaffolded and queued for full write-ups:

- **Files:** `cp`, `mv`, `rm`, `mkdir`, `touch`, `ln`
- **Navigation:** `pwd`, `man`, `alias`
- **Networking:** `ssh`, `scp`, `curl`, `wget`, `ping`, `netstat`
- **Permissions:** `chmod`, `chown`, `sudo`
- **Processes:** `ps`, `top`, `htop`, `kill`, `systemctl`
- **Disk:** `df`, `du`, `lsblk`
- **Archives:** `unzip` / `zip`
- **System:** `uname`, `uptime`, `env`, `history`, `cron` / `crontab`
- **Text Processing:** `sed`, `sort`, `wc`, `tee`, `diff`, `pip`

---

## 🗂️ Repository structure

```
linux-command-handbook/
├── README.md
├── LICENSE
└── commands/
    ├── navigation/
    │   └── <command>/
    │       ├── README.md
    │       ├── examples.md
    │       ├── edge-cases.md
    │       └── interview-questions.md
    ├── files/
    ├── text-processing/
    ├── search/
    ├── networking/
    ├── permissions/
    ├── processes/
    ├── disk/
    ├── archives/
    ├── system/
    └── user/
```

---

## 🤝 Contributing

Contributions are very welcome — this project grows faster with more hands on it. To add or improve a command page:

1. Pick a command from the **Coming soon** list above (or propose a new one via an issue).
2. Use one of the completed pages (e.g. [`awk`](commands/text-processing/awk/README.md) or [`find`](commands/search/find/README.md)) as a template for depth and structure.
3. Fill in all four files: `README.md`, `examples.md`, `edge-cases.md`, `interview-questions.md`.
4. Open a pull request — even partial or draft contributions are appreciated.

Found a typo, a broken example, or an outdated explanation? Open an issue or PR — keeping existing pages accurate matters as much as adding new ones.

---

## 🎯 Goals

- Learn Linux commands through practical, real-world examples
- Understand the edge cases that documentation usually skips
- Build a reference solid enough for daily use
- Prepare for technical interviews with command-specific Q&A

---

## 📜 License

MIT — see [LICENSE](LICENSE) for details.
