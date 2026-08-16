<div align="center">

# 🐧 Linux Command Handbook

**A practical, in-depth Linux command reference** — built one command at a time,
with real-world examples, the edge cases documentation usually skips, and
interview-ready explanations.

![License](https://img.shields.io/badge/license-CC%20BY%204.0-lightgrey.svg)
![Status](https://img.shields.io/badge/status-actively%20maintained-success.svg)
![Progress](https://img.shields.io/badge/commands-49%20%2F%2050-yellow.svg)
![PRs](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)

[Browse commands](#-completed-guides) · [What's inside](#-what-every-command-page-covers) · [Coming soon](#-coming-soon) 

</div>

---

This isn't a copy-paste cheat sheet. Every finished page covers what a command
actually **is**, how it works internally, its full syntax, dozens of
real-world examples, the edge cases that trip people up, and the kind of
questions you'd actually be asked about it in an interview.

## 🚧 Project status

This handbook is a **living project** — new command pages are added
regularly, and any existing page can be expanded or corrected over time.
Nothing here is ever "final." If a command isn't documented yet, it's most
likely already scaffolded and queued to be written next.

⭐ **Star or watch this repo** to follow new additions, and open an issue to
suggest or request a command.

<div align="center">

### Progress: 49 / 50 commands completed — 98%

`███████████████████████████████████████████░`

</div>

| Category | Progress | Status |
|---|:---:|:---:|
| 📁 Files | 5 / 6 | 🚧 In progress |
| 🔎 Search | 2 / 2 | ✅ Complete |
| 👤 User Management | 3 / 3 | ✅ Complete |
| 📝 Text Processing | 9 / 9 | ✅ Complete |
| 🧭 Navigation | 4 / 4 | ✅ Complete |
| 💽 Disk | 4 / 4 | ✅ Complete |
| 📦 Archives | 2 / 2 | ✅ Complete |
| 🌐 Networking | 6 / 6 | ✅ Complete |
| 🔐 Permissions | 3 / 3 | ✅ Complete |
| ⚙️ Processes | 5 / 5 | ✅ Complete |
| 🛠️ System | 6 / 6 | ✅ Complete |

---

## 📖 What every command page covers

Each command lives in its own folder — `commands/<category>/<command>/` —
and follows the same four-file format:

| File | Purpose |
|---|---|
| `README.md` | The complete reference — what it is, how it works internally, full syntax, all options, related commands |
| `examples.md` | Real-world, copy-pasteable usage examples |
| `edge-cases.md` | Gotchas, quirks, and situations that don't behave the way you'd expect |
| `interview-questions.md` | Common interview questions about the command, each with a full answer |

---

## ✅ Completed guides

<details open>
<summary><strong>🧭 Navigation</strong> — 4 / 4</summary>
<br>

| Command | Description |
|---|---|
| [`cd`](commands/navigation/cd/README.md) | Change the current directory |
| [`ls`](commands/navigation/ls/README.md) | List directory contents |
| [`pwd`](commands/navigation/pwd/README.md) | Print the working directory |
| [`man`](commands/navigation/man/README.md) | Display manual pages |

</details>

<details open>
<summary><strong>🔎 Search</strong> — 2 / 2</summary>
<br>

| Command | Description |
|---|---|
| [`grep`](commands/search/grep/README.md) | Search text using patterns |
| [`find`](commands/search/find/README.md) | Search for files in a directory hierarchy |

</details>

<details open>
<summary><strong>📝 Text Processing</strong> — 9 / 9</summary>
<br>

| Command | Description |
|---|---|
| [`cat`](commands/text-processing/cat/README.md) | Concatenate and display file contents |
| [`less` / `more`](commands/text-processing/less-more/README.md) | Page through file contents |
| [`sed`](commands/text-processing/sed/README.md) | Stream text editor |
| [`awk`](commands/text-processing/awk/README.md) | Pattern scanning and text-processing language |
| [`sort`](commands/text-processing/sort/README.md) | Sort lines of text |
| [`wc`](commands/text-processing/wc/README.md) | Count lines, words, and bytes in a stream |
| [`diff`](commands/text-processing/diff/README.md) | Compare file changes |
| [`tee`](commands/text-processing/tee/README.md) | Duplicate command output |
| [`pipe`](commands/text-processing/pipe/README.md) | Connect command outputs together |

</details>

<details open>
<summary><strong>💽 Disk</strong> — 4 / 4</summary>
<br>

| Command | Description |
|---|---|
| [`df`](commands/disk/df/README.md) | Report filesystem disk space usage |
| [`du`](commands/disk/du/README.md) | Estimate disk usage of files/directories |
| [`lsblk`](commands/disk/lsblk/README.md) | List block devices |
| [`mount`](commands/disk/mount/README.md) | Mount filesystems |

</details>

<details open>
<summary><strong>📦 Archives</strong> — 2 / 2</summary>
<br>

| Command | Description |
|---|---|
| [`tar`](commands/archives/tar/README.md) | Archive and compress files |
| [`zip` / `unzip`](commands/archives/unzip-zip/README.md) | Compress and extract files |

</details>

<details open>
<summary><strong>👤 User Management</strong> — 3 / 3</summary>
<br>

| Command | Description |
|---|---|
| [`useradd`](commands/user/useradd/README.md) | Create a new user account |
| [`passwd`](commands/user/passwd/README.md) | Change a user's password |
| [`who` / `w`](commands/user/who-w/README.md) | Show who is logged in |

</details>

<details open>
<summary><strong>🔐 Permissions</strong> — 3 / 3</summary>
<br>

| Command | Description |
|---|---|
| [`chmod`](commands/permissions/chmod/README.md) | Change file permissions |
| [`chown`](commands/permissions/chown/README.md) | Change file ownership |
| [`sudo`](commands/permissions/sudo/README.md) | Run a command as superuser |

</details>

<details open>
<summary><strong>⚙️ Processes</strong> — 5 / 5</summary>
<br>

| Command | Description |
|---|---|
| [`ps`](commands/processes/ps/README.md) | Snapshot of running processes |
| [`top`](commands/processes/top/README.md) | Live, continuously refreshing process monitor |
| [`htop`](commands/processes/htop/README.md) | Interactive, colorized process monitor |
| [`kill`](commands/processes/kill/README.md) | Send signals to a process |
| [`systemctl`](commands/processes/systemctl/README.md) | Control systemd services |

</details>

<details open>
<summary><strong>🌐 Networking</strong> — 6 / 6</summary>
<br>

| Command | Description |
|---|---|
| [`ping`](commands/networking/ping/README.md) | Test host reachability |
| [`curl`](commands/networking/curl/README.md) | Transfer data via URLs |
| [`wget`](commands/networking/wget/README.md) | Reliably download files |
| [`ssh`](commands/networking/ssh/README.md) | Secure remote shell |
| [`scp`](commands/networking/scp/README.md) | Securely copy files over SSH |
| [`netstat`](commands/networking/netstat/README.md) | Show network connections and listening ports |

</details>

<details open>
<summary><strong>🛠️ System</strong> — 6 / 6</summary>
<br>

| Command | Description |
|---|---|
| [`uname`](commands/system/uname/README.md) | Print system information |
| [`uptime`](commands/system/uptime/README.md) | Show how long the system has been running |
| [`env`](commands/system/env/README.md) | Display or modify the environment |
| [`history`](commands/system/history/README.md) | Display command history |
| [`alias`](commands/system/alias/README.md) | Create command shortcuts |
| [`cron` / `crontab`](commands/system/cron-crontab/README.md) | Automated background scheduling |

</details>

<details open>
<summary><strong>📁 Files</strong> — 5 / 6</summary>
<br>

| Command | Description |
|---|---|
| [`mkdir`](commands/files/mkdir/README.md) | Create new directories |
| [`touch`](commands/files/touch/README.md) | Creates or updates timestamps |
| [`rm`](commands/files/rm/README.md) | Deletes files permanently |
| [`cp`](commands/files/cp/README.md) | Copies files/directories |
| [`mv`](commands/files/mv/README.md) | Renames or relocates files |

</details>

---

## ⏳ Coming soon

Already scaffolded and queued for a full write-up:

`ln`

---

## 🗂️ Repository structure

```
linux-command-handbook/
├── README.md
├── LICENSE
└── commands/
    ├── navigation/
    ├── search/
    ├── text-processing/
    ├── disk/
    ├── archives/
    ├── user/
    ├── permissions/
    ├── processes/
    ├── networking/
    ├── system/
    └── files/
        └── <command>/
            ├── README.md
            ├── examples.md
            ├── edge-cases.md
            └── interview-questions.md
```

---

## 🎯 Goals

- Learn Linux commands through practical, real-world examples
- Understand the edge cases that documentation usually skips
- Build a reference solid enough for daily use
- Prepare for technical interviews with command-specific Q&A

## 📜 License

Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) —
see [LICENSE](LICENSE). You're free to share and adapt this content, even
commercially, as long as you give appropriate credit.
