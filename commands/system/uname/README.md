# uname — The Complete Reference

> **Print system identification information**
> Present since early Unix (standardized in POSIX), and one of the first
> commands anyone runs to answer "what kind of machine am I actually on?"

---

## Table of Contents

- [What is uname?](#what-is-uname)
- [Where does uname live?](#where-does-uname-live)
- [How uname works internally](#how-uname-works-internally)
- [Syntax](#syntax)
- [Understanding the Output](#understanding-the-output)
- [Kernel Name vs Kernel Release vs Kernel Version](#kernel-name-vs-kernel-release-vs-kernel-version)
- [Machine Hardware Name vs Processor vs Platform](#machine-hardware-name-vs-processor-vs-platform)
- [All Key Options](#all-key-options)
- [uname and /proc/version](#uname-and-procversion)
- [uname vs hostnamectl vs lsb_release vs /etc/os-release](#uname-vs-hostnamectl-vs-lsb_release-vs-etcos-release)
- [Related Commands](#related-commands)

---

## What is uname?

`uname` prints basic identifying information about the system it's run on: the kernel name, the machine's hostname, the kernel release and version, the hardware architecture, and the operating system name. Run with no options, it prints just the kernel name; run with `-a`, it prints everything it knows in one line.

```bash
uname -a
# Linux myhost 6.8.0-31-generic #31-Ubuntu SMP PREEMPT_DYNAMIC x86_64 x86_64 x86_64 GNU/Linux
```

**Why `uname` is often one of the first commands run on an unfamiliar system:** it instantly answers "what kernel and architecture am I dealing with?" — critical context before installing software, compiling code, choosing a binary release, or troubleshooting a compatibility issue.

---

## Where does uname live?

```
/usr/bin/uname
```

```bash
which uname
uname --version
# uname (GNU coreutils) 9.4
```

Part of **GNU coreutils** on most Linux distributions. Also present (with a smaller, POSIX-baseline option set) on macOS, BSD, and virtually every other Unix-like system — it's one of the oldest, most universally portable commands in existence.

---

## How uname works internally

`uname` is a thin wrapper around a single system call: `uname(2)`, which the kernel answers directly by filling in a `struct utsname` with values it already tracks (kernel name, hostname, release, version, machine architecture, and domain name).

```c
struct utsname {
    char sysname[];    // e.g. "Linux"
    char nodename[];   // hostname
    char release[];    // e.g. "6.8.0-31-generic"
    char version[];    // build date/flags string
    char machine[];    // e.g. "x86_64"
    char domainname[]; // NIS/YP domain name (usually "(none)")
};
```

`uname` doesn't compute, probe hardware, or read configuration files for most of its fields — it just formats whatever the kernel returns from that one syscall. (The one common exception: the operating system *name* string shown by `-o`, which on Linux is usually a static "GNU/Linux" label, not something the kernel computes.)

```bash
strace -e trace=uname uname -a
# uname({sysname="Linux", nodename="myhost", ...}) = 0
```

---

## Syntax

```bash
uname [OPTION]...
```

With no options, `uname` behaves as if `-s` had been given (kernel name only).

---

## Understanding the Output

```bash
uname -a
# Linux myhost 6.8.0-31-generic #31-Ubuntu SMP PREEMPT_DYNAMIC Thu Apr 25 12:00:00 UTC 2024 x86_64 x86_64 x86_64 GNU/Linux
```

| Segment | Meaning |
|---|---|
| `Linux` | Kernel name (`-s`) |
| `myhost` | Network hostname (`-n`) |
| `6.8.0-31-generic` | Kernel release (`-r`) |
| `#31-Ubuntu SMP PREEMPT_DYNAMIC Thu Apr 25 12:00:00 UTC 2024` | Kernel version — build number and build date, not a "product version" (`-v`) |
| `x86_64` (1st) | Machine hardware name (`-m`) |
| `x86_64` (2nd) | Processor type — on Linux, GNU coreutils just repeats the machine hardware name here, since a genuinely distinct value isn't tracked (`-p`) |
| `x86_64` (3rd) | Hardware platform — same story, usually identical to machine/processor on Linux (`-i`) |
| `GNU/Linux` | Operating system name (`-o`) |

---

## Kernel Name vs Kernel Release vs Kernel Version

These three are easy to conflate but mean distinct things:

```bash
uname -s
# Linux                          ← kernel NAME: which kernel family this is

uname -r
# 6.8.0-31-generic               ← kernel RELEASE: the specific version
#                                   number of that kernel, used for things
#                                   like matching kernel module directories
#                                   under /lib/modules/<release>/

uname -v
# #31-Ubuntu SMP PREEMPT_DYNAMIC Thu Apr 25 12:00:00 UTC 2024
#                                ← kernel VERSION: build metadata — a
#                                   distro-assigned build number and the
#                                   date/time it was compiled, NOT a
#                                   simple incrementing "version 31" in
#                                   the way software releases usually work
```

`uname -r`'s output is the one most commonly used programmatically — for example, to locate the matching kernel headers or modules directory for building out-of-tree drivers.

---

## Machine Hardware Name vs Processor vs Platform

```bash
uname -m
# x86_64     ← the actual CPU architecture the kernel was built for;
#               this is the one that matters for "will this binary run
#               here" questions (compare against a downloaded package's
#               architecture tag)

uname -p
# x86_64     ← "processor type"; on Linux this is essentially always
#               identical to -m, since Linux doesn't track a separate,
#               more specific processor identifier the way some other
#               Unix variants historically did

uname -i
# x86_64     ← "hardware platform"; likewise usually identical to -m
#               on Linux

# Common architecture values you'll encounter:
#   x86_64   — 64-bit Intel/AMD (most desktops, servers)
#   aarch64  — 64-bit ARM (Raspberry Pi 4/5, Apple Silicon under Linux, many phones/servers)
#   armv7l   — 32-bit ARM (older Raspberry Pi models, embedded devices)
#   i686     — 32-bit x86 (legacy)
```

---

## All Key Options

| Option | Long form | Description |
|---|---|---|
| (none) | — | Same as `-s` — kernel name only |
| `-a` | `--all` | Print all available information, in a fixed order |
| `-s` | `--kernel-name` | Kernel name (e.g., `Linux`) |
| `-n` | `--nodename` | Network hostname |
| `-r` | `--kernel-release` | Kernel release string |
| `-v` | `--kernel-version` | Kernel version/build string |
| `-m` | `--machine` | Machine hardware architecture (e.g., `x86_64`) |
| `-p` | `--processor` | Processor type (often identical to `-m` on Linux) |
| `-i` | `--hardware-platform` | Hardware platform (often identical to `-m` on Linux) |
| `-o` | `--operating-system` | Operating system name (e.g., `GNU/Linux`) |
| — | `--help` | Print usage help |
| — | `--version` | Print version information |

> Portability note: `-p`, `-i`, and `-o` are **GNU coreutils extensions**. The POSIX-baseline `uname` (as found on macOS/BSD) supports only `-s -n -r -v -m` and `-a` — scripts intended to run on macOS should not rely on `-p`, `-i`, or `-o` being present.

---

## uname and /proc/version

```bash
cat /proc/version
# Linux version 6.8.0-31-generic (buildd@lcy02-amd64-119) (x86_64-linux-gnu-gcc-13 ...) #31-Ubuntu SMP PREEMPT_DYNAMIC Thu Apr 25 12:00:00 UTC 2024
```

`/proc/version` carries much the same information as `uname -a`, plus extra detail `uname` doesn't expose at all — notably the **compiler** used to build the running kernel. Useful when the compiler toolchain version matters (building kernel modules that must match exactly).

---

## uname vs hostnamectl vs lsb_release vs /etc/os-release

| Tool | Best for | Key difference from uname |
|---|---|---|
| `uname` | Kernel name/release/version, CPU architecture | Says nothing about which *distribution* is installed — only about the kernel and hardware |
| `hostnamectl` | Hostname plus a friendlier system summary (systemd systems) | Also shows OS pretty-name, kernel, virtualization type, and architecture together in one readable block |
| `lsb_release -a` | Distribution name, version, and codename | Distro-focused; historically the standard way to identify "which Linux distro," though increasingly superseded |
| `cat /etc/os-release` | Machine-readable distro identification | The modern, standardized way scripts should detect the distro (`ID`, `VERSION_ID`, `PRETTY_NAME` fields) — `uname` cannot tell you this at all |

```bash
uname -a               # "what kernel, architecture, and hostname?"
cat /etc/os-release     # "which distribution and version?" — uname doesn't know this
hostnamectl              # "give me a friendly one-shot summary of both"
```

**Key takeaway:** `uname` only ever reports on the **kernel**, never the distribution built on top of it. Two very different distros (Ubuntu and Fedora, for instance) running the same kernel release will report identical `uname -a` output aside from hostname — distro identity always requires a separate source like `/etc/os-release`.

---

## Related Commands

| Command | Relation |
|---|---|
| `hostname` | Shows/sets just the network hostname — a narrower version of `uname -n` |
| `hostnamectl` | Friendlier combined summary including OS pretty-name and virtualization |
| `cat /etc/os-release` | The standard way to identify distribution name/version (which `uname` cannot do) |
| `lscpu` | Much more detailed CPU information than `uname -m`'s single architecture string |
| `arch` | Legacy shorthand, equivalent to `uname -m` |
| `cat /proc/version` | Kernel build string plus compiler info, a superset of `uname -v` |
| `dpkg --print-architecture` / `rpm --eval '%{_arch}'` | Distro package-manager's own architecture label, sometimes phrased differently than `uname -m` (e.g., `amd64` vs `x86_64`) |

---

> See also: [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
