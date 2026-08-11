# wget — Interview Questions

> Covers beginner to senior level. Each question includes the answer.
> Questions marked 🔥 are commonly asked in real interviews.

---

## Table of Contents

- [Conceptual](#conceptual)
- [Resuming and Retrying](#resuming-and-retrying)
- [Recursive Downloading](#recursive-downloading)
- [Internals](#internals)
- [wget vs Other Tools](#wget-vs-other-tools)
- [Scenario-Based](#scenario-based)

---

## Conceptual

**Q1. What does wget do, and what does its name reflect?**
> Downloads files over HTTP, HTTPS, and FTP. Its name ("web get") reflects its original core design goal: reliable, non-interactive operation — working correctly in the background without a person watching it, including automatic retry and resume behavior.

---

**Q2 🔥 What does wget do by default when run with just a URL and no other options?**
> Downloads the resource, automatically deriving a local filename from the URL's path, and shows a live progress bar in the terminal while saving the content directly to that file.

---

**Q3. Is wget preinstalled on macOS by default?**
> No — unlike curl, which ships built into macOS, wget must be installed separately (e.g., via Homebrew). This is a notable portability difference to keep in mind for cross-platform scripts.

---

## Resuming and Retrying

**Q4 🔥 What does the -c flag do?**
> Resumes a partially downloaded file instead of restarting from scratch — wget checks how much of the local file already exists and requests only the remaining bytes from the server via an HTTP Range request.

---

**Q5. What's a requirement for -c to actually work as intended?**
> The remote server must support HTTP Range requests for that resource. If it doesn't, wget can't cleanly resume and may behave differently depending on version and the server's actual response, rather than achieving a genuine partial resume.

---

**Q6 🔥 What does `--tries=0` do?**
> Tells wget to retry indefinitely on failure rather than giving up after a fixed number of attempts — useful for long-running, unattended jobs where eventual success matters more than a bounded number of retries.

---

## Recursive Downloading

**Q7. What does the -r flag enable, and what does wget lack that curl also lacks in this area?**
> `-r` enables recursive downloading — wget follows links discovered on fetched pages and downloads them too, applying configurable depth/domain restrictions. curl has no native equivalent to this recursive capability at all; it's one of wget's clearer differentiators.

---

**Q8 🔥 Why is it important to combine -r with constraints like -l, --no-parent, or --domains?**
> Without constraints, a recursive download can follow links far beyond what was intended — including into entirely different domains or effectively unbounded depth — consuming excessive bandwidth, disk space, and time. These flags keep the crawl scoped to what was actually intended.

---

**Q9. Does wget respect a site's robots.txt rules during a recursive download by default?**
> Yes — by default, wget checks and honors robots.txt restrictions, silently skipping disallowed paths during a crawl. This can result in a seemingly incomplete mirror with no loud error explaining the gap, since the skipping happens quietly and by design.

---

## Internals

**Q10 🔥 Where does wget send its progress/status information by default, versus where the downloaded content goes?**
> Status output (resolving, connecting, response codes, the progress bar) goes to stderr by default, while the actual downloaded content is written directly to the output file — the opposite of curl's default of printing the response body to stdout.

---

**Q11. What happens if you run the same wget download command twice in a row, targeting the same output filename, without any special flags?**
> By default, wget never overwrites an existing file with the same name — it appends an incrementing numeric suffix instead (`file.pdf`, then `file.pdf.1`, then `file.pdf.2`, and so on) on each repeated run.

---

**Q12 🔥 How would you make wget always overwrite a file rather than creating numbered duplicates, or alternatively skip re-downloading if it already exists?**
> `-O filename` forces a specific output name and overwrites it directly on each run. `-nc` ("no clobber") does the opposite — it skips the download entirely if a local file with that name already exists, rather than creating a numbered duplicate.

---

## wget vs Other Tools

**Q13 🔥 When might you choose wget over curl for a download task?**
> When the task is primarily "reliably download and save this file (or site)" — wget's defaults are already oriented toward that outcome (automatic filename derivation, resumable transfers, native recursive/mirroring support) with less explicit flag composition needed than achieving the same result with curl.

---

**Q14. What does aria2c offer that plain wget doesn't?**
> Multi-connection, segmented downloading — splitting a single file's download across several simultaneous connections to maximize throughput. wget downloads a given file as one single stream, which can be slower on high-bandwidth connections that could otherwise support more parallelism.

---

## Scenario-Based

**Q15 🔥 A teammate runs `wget https://api.example.com/download/12345` and the resulting file, despite having no obvious errors shown, turns out to just be an HTML error page when opened. What went wrong, and how would you catch this earlier?**
> wget saves whatever content the server actually returns, without automatically verifying it matches the expected file type — if the endpoint returned an HTML error/login page instead of the real binary, that gets saved exactly as if it were the intended download. Running `file <downloaded_file>` immediately after, or checking wget's own reported HTTP status/exit code, would have caught the mismatch before assuming success.

---

**Q16. A cron job runs the same `wget https://example.com/report.pdf` command daily, and over a month the directory has accumulated dozens of files named report.pdf, report.pdf.1, report.pdf.2, etc. What's causing this, and what's the fix?**
> wget never overwrites an existing file with the same name by default — each repeated run creates a new numbered variant instead. The fix is either `-O report.pdf` to force overwriting the same filename each time, or `-nc` if the intent was actually to skip re-downloading when a copy already exists.

---

**Q17 🔥 A recursive `wget -r` command intended to grab one documentation subsection ends up downloading a huge portion of the entire website instead. What's the most likely missing safeguard?**
> Missing constraints — most likely `--no-parent` (which would have prevented the crawl from ascending to parent directories via any "back to home" or breadcrumb-style links) and/or a depth limit (`-l`) or domain restriction (`--domains`). Without these, a recursive crawl follows every discovered link outward with no natural boundary.

---

**Q18. During a security review, someone flags a script that runs `wget --user=alice --password=supersecret https://...` as a concern. What's the issue, and what's the recommended fix?**
> Command-line arguments (including the password) are visible to other users on the same system via process listing tools while the command runs, and often end up preserved in shell history afterward — neither is a secure way to handle a credential. The recommended fix is a `.netrc`-style credential file with restrictive permissions (`chmod 600`), which wget can read from instead of exposing the secret directly on the command line.

---

**Q19 🔥 A batch download via `wget -i urls.txt` completes with an overall exit code of 0, but a teammate later discovers a few of the individual files in the list never actually downloaded successfully. How is this possible, and what should the script have checked instead?**
> Depending on version and the specific failure mode, an overall "success" exit code from a multi-file batch job doesn't always guarantee every individual URL in the list succeeded — some per-file failures can be logged without necessarily flipping the final overall exit code. A more reliable script verifies the actual output for each expected file (or counts successful saves in the log against the total URL count) rather than trusting a single overall exit code for the whole batch.

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md)
