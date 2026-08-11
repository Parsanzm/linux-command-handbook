# wget — Practical Examples

> Real-world patterns for downloading, resuming, mirroring, and
> automating unattended transfers.

---

## Table of Contents

- [Basic Usage](#basic-usage)
- [Saving with a Different Name or Location](#saving-with-a-different-name-or-location)
- [Resuming and Retrying](#resuming-and-retrying)
- [Downloading Multiple Files](#downloading-multiple-files)
- [Mirroring a Site](#mirroring-a-site)
- [Background and Unattended Downloads](#background-and-unattended-downloads)
- [Real-World Recipes](#real-world-recipes)

---

## Basic Usage

```bash
# Basic download, filename derived automatically from the URL
wget https://example.com/file.zip

# Quiet mode — no progress bar or status chatter
wget -q https://example.com/file.zip

# Download and show output to stdout instead of saving to a file
wget -O - https://example.com/file.zip
```

---

## Saving with a Different Name or Location

```bash
# Save under a specific filename
wget -O archive.zip https://example.com/download?id=123

# Save into a specific directory
wget -P /tmp/downloads https://example.com/file.zip

# Combine both
wget -P /tmp/downloads -O renamed.zip https://example.com/file.zip
```

---

## Resuming and Retrying

```bash
# Resume an interrupted download
wget -c https://example.com/largefile.iso

# Retry up to 10 times on failure, waiting 5 seconds between attempts
wget --tries=10 --waitretry=5 https://example.com/largefile.iso

# Retry indefinitely — useful for a long-running unattended job
wget --tries=0 -c https://example.com/largefile.iso

# Set a timeout so a hung connection doesn't stall forever
wget --timeout=30 https://example.com/largefile.iso
```

---

## Downloading Multiple Files

```bash
# Download a list of URLs from a text file, one per line
wget -i urls.txt

# Download several files given directly on the command line
wget https://example.com/file1.zip https://example.com/file2.zip

# Download all files matching a pattern from a directory listing (FTP)
wget -r -A "*.csv" ftp://ftp.example.com/data/
```

---

## Mirroring a Site

```bash
# Full local mirror, browsable offline
wget --mirror --convert-links --page-requisites --no-parent https://example.com/docs/

# Limit how deep the recursion goes
wget -r -l 3 https://example.com/docs/

# Restrict to a single domain, avoiding following external links
wget -r --domains=example.com --no-parent https://example.com/

# Pace the crawl to avoid overwhelming the target server
wget -r --wait=1 --random-wait https://example.com/docs/
```

---

## Background and Unattended Downloads

```bash
# Start in the background immediately, logging to wget-log
wget -b https://example.com/largefile.iso

# Follow the background job's progress
tail -f wget-log

# A background download that survives an ssh session ending, kicked
# off remotely and left running unattended
ssh alice@server "wget -b -c https://example.com/largefile.iso"
```

---

## Real-World Recipes

```bash
# --- Download and Verify a File's Integrity ---
wget https://example.com/release.tar.gz
wget https://example.com/release.tar.gz.sha256
sha256sum -c release.tar.gz.sha256

# --- Resumable Overnight Download of a Large Dataset ---
wget -c --tries=0 --waitretry=30 -P /data/downloads \
  https://example.com/huge-dataset.tar.gz

# --- Mirror Documentation for Offline Reading ---
wget --mirror --convert-links --page-requisites --no-parent \
  -P ./offline-docs https://example.com/docs/

# --- Fetch a List of Report Files Overnight ---
wget -i report_urls.txt -P ./reports --tries=5 --waitretry=10

# --- Rate-Limited Download to Avoid Saturating a Shared Connection ---
wget --limit-rate=500k https://example.com/largefile.iso

# --- Download Behind Basic Auth ---
wget --user=alice --password=secret https://example.com/protected/file.zip
```

---

> See also: [`README.md`](README.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
