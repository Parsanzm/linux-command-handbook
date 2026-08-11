# wget — The Complete Reference

> **Non-interactive network downloader**
> Built from the ground up for retrieving files reliably — including
> resuming interrupted downloads and mirroring entire sites — without
> needing a terminal session to stay open and watch it.

---

## Table of Contents

- [What is wget?](#what-is-wget)
- [Where does wget live?](#where-does-wget-live)
- [How wget works internally](#how-wget-works-internally)
- [Syntax](#syntax)
- [Understanding the Output](#understanding-the-output)
- [Resuming and Retrying Downloads](#resuming-and-retrying-downloads)
- [Recursive Downloading and Mirroring](#recursive-downloading-and-mirroring)
- [All Key Options](#all-key-options)
- [Background Downloads and Logging](#background-downloads-and-logging)
- [wget vs curl vs aria2c](#wget-vs-curl-vs-aria2c)
- [Related Commands](#related-commands)

---

## What is wget?

`wget` ("web get") downloads files from HTTP, HTTPS, and FTP servers. The name reflects its original design goal — **non-interactive** operation: it's built to keep working reliably in the background, over flaky connections, without a person watching it, including automatically retrying and resuming after a dropped connection.

```bash
wget https://example.com/file.zip
# --2026-08-11 14:32:00--  https://example.com/file.zip
# Resolving example.com (example.com)... 93.184.216.34
# Connecting to example.com (example.com)|93.184.216.34|:443... connected.
# HTTP request sent, awaiting response... 200 OK
# Length: 10485760 (10M) [application/zip]
# Saving to: 'file.zip'
# file.zip    100%[===================>]  10.00M  5.23MB/s    in 1.9s
```

**Why wget remains a go-to tool despite curl's broader popularity:** its defaults are already oriented around "download and save reliably" — sensible filenames derived automatically, resumable transfers, and built-in recursive/mirroring capability — without needing as much explicit flag composition as achieving the same result in curl.

---

## Where does wget live?

```
/usr/bin/wget
```

```bash
which wget
wget --version
# GNU Wget 1.21.4 built on linux-gnu.
```

Part of the **GNU Wget** project. Commonly preinstalled on most Linux distributions, though not universally guaranteed on every minimal image; **not preinstalled on macOS by default** (requires Homebrew or similar), unlike `curl` which ships with macOS out of the box.

---

## How wget works internally

For an HTTP(S) URL, `wget` performs largely the same underlying steps as `curl` — DNS resolution, TCP connection, TLS handshake for HTTPS, sending the request, and reading the response — but its defaults and internal design specifically optimize for **saving the result reliably to disk**, rather than curl's design center of printing to stdout for pipeline use.

```bash
# wget shows its own progress/status to stderr by default, and saves
# the actual response BODY directly to a file — the inverse of curl's
# "body to stdout by default" behavior
wget -O - https://example.com
# -O - forces output to stdout instead, mimicking curl's default
```

For recursive/mirroring operations, `wget` parses the HTML (or FTP directory listing) of each fetched page to discover further links, applying configurable depth, domain, and file-type restrictions to control how far and wide it follows them.

---

## Syntax

```bash
wget [OPTIONS] URL
```

Run with just a URL and no other options, `wget` downloads the resource, automatically deriving a local filename from the URL's path, and shows a live progress bar on the terminal.

---

## Understanding the Output

```bash
wget https://example.com/file.zip
```

| Line | Meaning |
|---|---|
| `Resolving example.com...` | DNS lookup for the hostname |
| `Connecting to ...` | TCP connection (and TLS handshake, for HTTPS) |
| `HTTP request sent, awaiting response... 200 OK` | The actual HTTP status returned |
| `Length: ... [content-type]` | Reported size and MIME type from the server, if provided |
| `Saving to: 'file.zip'` | The local filename wget derived and will write to |
| Progress bar | Live percentage, transfer rate, and ETA |

By default, all of this status information goes to **stderr**, while the downloaded content itself is written to the output file — meaning wget's normal terminal chatter doesn't interfere even if the destination is somehow piped elsewhere.

---

## Resuming and Retrying Downloads

```bash
# Resume a previously interrupted download instead of starting over
wget -c https://example.com/largefile.iso
# -c ("continue") checks how much of the local file already exists
# and requests only the REMAINING bytes from the server (via an HTTP
# Range header), rather than re-downloading everything from scratch

# Automatically retry on a failed/dropped connection
wget --tries=5 https://example.com/largefile.iso

# Retry indefinitely until it succeeds (useful for scripted, unattended jobs)
wget --tries=0 https://example.com/largefile.iso

# Wait between retries, useful for avoiding hammering a struggling server
wget --waitretry=10 https://example.com/largefile.iso
```

This resume/retry behavior is central to wget's original design purpose — reliably completing large downloads over unreliable connections without needing to babysit the process.

---

## Recursive Downloading and Mirroring

```bash
# Download an entire site (or a subsection of it), following links
wget -r https://example.com/docs/

# A full, faithful mirror — preserves timestamps, converts links for
# local browsing, keeps the directory structure intact
wget --mirror --convert-links --page-requisites --no-parent https://example.com/docs/

# Limit recursion depth
wget -r -l 2 https://example.com/docs/

# Only follow links within the same domain
wget -r --domains=example.com https://example.com/docs/

# Download only specific file types recursively (e.g., all PDFs on a site)
wget -r -A pdf https://example.com/reports/
```

This built-in recursive/mirroring capability, without needing external scripting to follow discovered links, is one of wget's clearest differentiators from curl, which has no native equivalent.

---

## All Key Options

| Option | Long form | Description |
|---|---|---|
| `-O FILE` | `--output-document=FILE` | Save to a specific filename (or `-O -` for stdout) |
| `-c` | `--continue` | Resume a partially downloaded file instead of restarting |
| `-b` | `--background` | Go to background immediately after starting |
| `-q` | `--quiet` | Suppress all output |
| `-v` | `--verbose` | Extra detail (the default level, actually — `-nv` for less) |
| `-r` | `--recursive` | Follow links and download recursively |
| `-l DEPTH` | `--level=DEPTH` | Maximum recursion depth |
| `--mirror` | | Shorthand for a common recursive-mirroring flag combination |
| `-np` | `--no-parent` | Never ascend to the parent directory when recursing |
| `-A LIST` | `--accept` | Only download files matching these extensions/patterns |
| `-R LIST` | `--reject` | Skip files matching these extensions/patterns |
| `--limit-rate=RATE` | | Cap the download speed (e.g., `--limit-rate=200k`) |
| `-U STRING` | `--user-agent=STRING` | Set a custom User-Agent header |
| `--tries=N` | | Number of retry attempts (`0` for unlimited) |
| `-i FILE` | `--input-file=FILE` | Read a list of URLs to download from a file |
| `-P DIR` | `--directory-prefix=DIR` | Save downloaded files under a specific directory |

---

## Background Downloads and Logging

```bash
# Start a download, then immediately move it to the background
wget -b https://example.com/largefile.iso
# Continuing in background, pid 12345.
# Output will be written to 'wget-log'.

# Follow the background download's progress live
tail -f wget-log

# Explicitly direct logging to a specific file instead of the default
wget -b -o download.log https://example.com/largefile.iso
```

Because `wget` is designed for unattended operation, `-b` (background) combined with its automatic logging is a common pattern for kicking off a large download over `ssh` and disconnecting without losing progress — no terminal multiplexer required for this specific use case.

---

## wget vs curl vs aria2c

| Tool | Best for | Key difference from wget |
|---|---|---|
| `wget` | Straightforward downloads, resumable transfers, site mirroring | Simpler defaults for "just download and save this," native recursive capability |
| `curl` | API testing, broader protocol support, scripting flexibility | Prints to stdout by default (pipeline-friendly), far more request-shaping options (custom methods, headers, auth), no native recursive downloading |
| `aria2c` | Maximum download speed via multi-connection/segmented downloading | Splits a single download across multiple simultaneous connections for higher throughput; wget downloads a file as one single stream |

```bash
wget -c https://example.com/file.zip       # simple, resumable download
curl -O https://example.com/file.zip         # equivalent one-shot download via curl
aria2c -x4 https://example.com/file.zip       # same download, 4 parallel connections
```

---

## Related Commands

| Command | Relation |
|---|---|
| `curl` | The more flexible, request-shaping-focused alternative for the same broad task |
| `aria2c` | Multi-connection downloader for maximizing transfer speed |
| `rsync` | Better suited for efficient, resumable synchronization when both ends support it |
| `httrack` | A dedicated website-mirroring tool with more specialized options than wget's `--mirror` |
| `curl -O` | The closest direct curl equivalent to a basic `wget` download |

---

> See also: [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
