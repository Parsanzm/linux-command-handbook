# wget — Edge Cases & Gotchas

> `wget` looks like a simple "download this URL" tool, but recursive
> mirroring, filename collisions, and default retry behavior routinely
> produce surprising results if used on autopilot.

---

## Table of Contents

- [Running wget on a URL Without a File Extension Can Save an HTML Error Page](#running-wget-on-a-url-without-a-file-extension-can-save-an-html-error-page)
- [Re-running wget on the Same File Creates .1, .2, .3 Copies Instead of Overwriting](#re-running-wget-on-the-same-file-creates-1-2-3-copies-instead-of-overwriting)
- [-c Silently Does Nothing Useful If the Server Doesn't Support Range Requests](#-c-silently-does-nothing-useful-if-the-server-doesnt-support-range-requests)
- [Recursive Downloads Can Explode Far Beyond What Was Intended](#recursive-downloads-can-explode-far-beyond-what-was-intended)
- [wget Respects robots.txt by Default — Which Can Silently Skip Content](#wget-respects-robotstxt-by-default--which-can-silently-skip-content)
- [Not Preinstalled on macOS by Default, Unlike curl](#not-preinstalled-on-macos-by-default-unlike-curl)
- [A 404 Response Still Gets "Saved" as a File Unless You Check](#a-404-response-still-gets-saved-as-a-file-unless-you-check)
- [--mirror Without --no-parent Can Crawl Well Beyond the Intended Directory](#--mirror-without---no-parent-can-crawl-well-beyond-the-intended-directory)
- [Passwords on the Command Line Are Visible to Anyone Checking Process Lists](#passwords-on-the-command-line-are-visible-to-anyone-checking-process-lists)
- [wget's Exit Code Can Be 0 Even When Individual Files in a Batch Failed](#wgets-exit-code-can-be-0-even-when-individual-files-in-a-batch-failed)
- [Redirect Chains Can Behave Unexpectedly with POST-Like Interactions](#redirect-chains-can-behave-unexpectedly-with-post-like-interactions)

---

## Running wget on a URL Without a File Extension Can Save an HTML Error Page

### The saved file isn't automatically verified as the "real" content
```bash
wget https://api.example.com/download/12345
# Saving to: '12345'
# ⚠️ If that endpoint actually returns an HTML error page (a 404, a
# login wall, an API error response) rather than the expected binary
# file, wget saves that HTML content exactly as if it were the real
# download — there's no automatic content-type sanity check comparing
# what was expected against what was actually received.

file 12345
# 12345: HTML document, ASCII text
# ⚠️ Only checking the saved file's actual type afterward reveals the
# mistake — always verify what was actually downloaded when the
# source URL doesn't have an obvious, self-describing file extension.
```

---

## Re-running wget on the Same File Creates .1, .2, .3 Copies Instead of Overwriting

### A frequent source of accumulating duplicate files
```bash
wget https://example.com/report.pdf
# Saving to: 'report.pdf'
wget https://example.com/report.pdf
# Saving to: 'report.pdf.1'
wget https://example.com/report.pdf
# Saving to: 'report.pdf.2'
# ⚠️ By default, wget NEVER overwrites an existing local file with
# the same name — it appends an incrementing suffix instead, on every
# repeated download. Running the same wget command in a loop or
# repeated cron job without realizing this can silently accumulate a
# large number of near-duplicate files over time.

# To always overwrite instead:
wget -O report.pdf https://example.com/report.pdf
# -O forces a specific output filename and DOES overwrite it directly

# Or to explicitly skip re-downloading if a local copy already exists:
wget -nc https://example.com/report.pdf
# -nc ("no clobber") skips the download entirely if the file already
# exists locally, rather than creating report.pdf.1
```

---

## -c Silently Does Nothing Useful If the Server Doesn't Support Range Requests

### Resume depends entirely on server-side support, which isn't guaranteed
```bash
wget -c https://example.com/largefile.iso
# ⚠️ -c requests a partial (Range) transfer starting from where the
# local file left off — but this ONLY works if the remote SERVER
# actually supports HTTP Range requests for that resource. If it
# doesn't, wget's behavior can vary: some servers return the FULL
# file again (which wget may then handle by starting over, or by
# producing a corrupted/mismatched result depending on version and
# exact server response), rather than cleanly resuming as expected.

# Confirm Range support is actually present before relying heavily on -c:
curl -I https://example.com/largefile.iso | grep -i accept-ranges
# Accept-Ranges: bytes    ← confirms genuine resume support is available
```

---

## Recursive Downloads Can Explode Far Beyond What Was Intended

### A missing constraint can turn "download this page" into "download most of the internet"
```bash
wget -r https://example.com/blog/post-1
# ⚠️ Without ANY constraints (depth limit, domain restriction, "no
# parent"), a recursive download follows EVERY discovered link
# outward — potentially including links to entirely OTHER domains,
# unrelated site sections, or effectively unbounded depth — consuming
# far more bandwidth, disk space, and time than intended, and
# potentially pulling in content well beyond what was actually wanted.

# Always constrain recursive downloads explicitly:
wget -r -l 2 --no-parent --domains=example.com https://example.com/blog/post-1
```

---

## wget Respects robots.txt by Default — Which Can Silently Skip Content

### Content can go missing from a recursive download without an obvious error
```bash
wget -r https://example.com/private-section/
# ⚠️ By default, wget CHECKS and RESPECTS a site's robots.txt rules
# during recursive crawls, silently skipping any paths that file
# disallows — this can result in a seemingly "incomplete" mirror with
# NO loud error explaining why certain expected pages/files never got
# downloaded, since robots.txt compliance happens quietly.

# To override this (only do so on sites you're authorized to crawl
# fully, respecting the site owner's actual wishes/terms):
wget -r -e robots=off https://example.com/private-section/
```

---

## Not Preinstalled on macOS by Default, Unlike curl

### A frequent surprise for anyone assuming wget is universally available
```bash
wget https://example.com/file.zip
# zsh: command not found: wget
# ⚠️ Unlike curl, which ships built into macOS by default, wget is
# NOT preinstalled on macOS — it needs to be installed separately
# (Homebrew, MacPorts, etc.) before use. Scripts intended to be
# cross-platform-portable shouldn't assume wget's presence the same
# way curl's presence can generally be assumed on both Linux and macOS.

brew install wget
```

---

## A 404 Response Still Gets "Saved" as a File... Unless wget's Own Status Check Catches It

### wget DOES report the HTTP status clearly, but only if actually watched/checked
```bash
wget https://example.com/does-not-exist.zip
# HTTP request sent, awaiting response... 404 Not Found
# 2026-08-11 14:32:00 ERROR 404: Not Found.
# ⚠️ Unlike curl (which returns exit code 0 for a 404 by default),
# wget DOES correctly report a non-zero exit status for an HTTP error
# by default — but a script that doesn't actually CHECK wget's exit
# code (assuming any file appearing on disk means success) can still
# miss the fact that an error page or empty output was saved instead
# of real content, if the response happened to include SOME body text
# alongside the error status.

wget https://example.com/does-not-exist.zip
echo $?
# 8    ← non-zero, correctly reflecting the server error — always
# check this in scripts rather than assuming presence-of-file means success
```

---

## --mirror Without --no-parent Can Crawl Well Beyond the Intended Directory

### The "mirror" shortcut is powerful, and easy to point too broadly
```bash
wget --mirror https://example.com/docs/getting-started/
# ⚠️ Without --no-parent specifically, a recursive/mirror crawl can
# ascend to PARENT directories relative to the starting URL if
# there's a link pointing "up" (a breadcrumb nav, a "back to home"
# link) — potentially mirroring the ENTIRE site rather than just the
# specific subsection that was actually intended.

wget --mirror --no-parent https://example.com/docs/getting-started/
# --no-parent keeps the crawl strictly within/below the starting path
```

---

## Passwords on the Command Line Are Visible to Anyone Checking Process Lists

### --password=... is a common but risky pattern
```bash
wget --user=alice --password=supersecret https://example.com/protected/file.zip
# ⚠️ Command-line arguments are visible to OTHER users on the same
# system via `ps aux` (or similar) while the command is running, and
# often end up preserved in shell history files afterward too —
# neither is a secure way to pass a genuine secret.

# Prefer a .wgetrc / .netrc-based credential file with restrictive
# permissions instead of exposing the password directly on the
# command line:
echo "machine example.com login alice password supersecret" > ~/.netrc
chmod 600 ~/.netrc
wget --auth-no-challenge https://example.com/protected/file.zip
```

---

## wget's Exit Code Can Be 0 Even When Individual Files in a Batch Failed

### Multi-file downloads via -i need per-file verification, not just an overall exit check
```bash
wget -i urls.txt
echo $?
# 0     ← ⚠️ depending on version and exact failure mode, an overall
# "success" exit code from a MULTI-file batch download (via -i) does
# not always guarantee that EVERY individual URL in the list actually
# succeeded — some failures partway through a batch can be reported
# in the log/output without necessarily flipping the FINAL overall
# exit code to non-zero in every situation.

# Verify the actual output for each expected file rather than trusting
# a single overall exit code for a multi-file batch job:
grep -c "saved" wget-log
wc -l urls.txt
# compare the counts to confirm everything actually succeeded
```

---

## Redirect Chains Can Behave Unexpectedly with POST-Like Interactions

### wget's redirect handling is primarily built around GET-style downloads
```bash
# wget is fundamentally designed around fetching/downloading resources
# (GET-style behavior) rather than the fuller general-purpose HTTP
# request shaping curl offers — POST support exists (--post-data) but
# is comparatively limited, and following a redirect after a POST-like
# interaction may not behave identically to a browser or to curl's own
# handling of the same scenario, depending on the specific redirect
# status code and server behavior involved.

# For anything beyond straightforward file downloading — especially
# API-style POST/PUT interactions with careful redirect semantics —
# curl is generally the more predictable, purpose-appropriate tool:
curl -L -X POST -d "key=value" https://example.com/endpoint
```

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`interview-questions.md`](interview-questions.md)
