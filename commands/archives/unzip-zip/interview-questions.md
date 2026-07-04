# zip / unzip — Interview Questions

> Covers beginner to senior level. Each question includes the answer.
> Questions marked 🔥 are commonly asked in real interviews.

---

## Table of Contents

- [Conceptual](#conceptual)
- [Basic Usage](#basic-usage)
- [Format Internals](#format-internals)
- [zip vs Other Formats](#zip-vs-other-formats)
- [Compression & Options](#compression--options)
- [Security](#security)
- [Scenario-Based](#scenario-based)

---

## Conceptual

**Q1. What's the fundamental structural difference between how zip and tar.gz store multiple files?**
> ZIP compresses each file **independently**, with its own header and compressed data block, plus a Central Directory index at the end listing every entry. `tar.gz` instead concatenates all files into a single **tar stream first**, then compresses that entire stream as one continuous unit with gzip. This is why ZIP allows extracting or listing a single file instantly, while tar.gz generally must be read from the beginning to reach a specific file.

---

**Q2 🔥 Why is zip's compression ratio usually a bit worse than tar.gz for a folder of many small, similar files?**
> Because each ZIP entry is compressed in isolation, the compressor can't exploit redundancy **between** different files — it only sees one file's bytes at a time. `tar.gz` compresses the whole concatenated stream together, so similar content repeated across multiple files (shared headers, boilerplate, similar structure) can be compressed away collectively, often producing a smaller total size for many small, similar files.

---

**Q3. What is the Central Directory in a ZIP file, and why does its location matter?**
> It's an index, stored at the **end** of the archive, listing every file entry along with where its data begins. Because it's at the end, tools can instantly list or seek to a specific file without scanning the whole archive — but it also means a **truncated download** (missing the end of the file) can make the entire archive appear unreadable, even if most of the actual file data earlier in the archive is intact.

---

## Basic Usage

**Q4 🔥 What's the difference between `zip archive.zip file.txt` and `zip -r archive.zip folder/`?**
> Without `-r`, zip only adds the files explicitly listed (and would just add an empty entry for a directory without descending into it). `-r` tells zip to recurse into a directory, adding every file and subdirectory inside it. `-r` is required whenever the target is a directory you want the contents of, not just the directory entry itself.

---

**Q5. How do you list the contents of a zip archive without extracting anything?**
> ```bash
> unzip -l archive.zip
> ```
> This reads only the Central Directory at the end of the file and prints filenames, sizes, and dates — it never touches the compressed data itself, which is why it's fast even on very large archives.

---

**Q6 🔥 How do you extract just one specific file from a large zip archive without extracting everything else?**
> ```bash
> unzip archive.zip "path/inside/archive/specific_file.txt"
> ```
> Because of the Central Directory index, unzip can seek directly to that file's data without reading through the rest of the archive first.

---

## Format Internals

**Q7. What are the two main compression methods a ZIP entry can use, and when is each chosen?**
> **Deflate** (method 8) is the standard, most common method for compressible content. **Stored** (method 0, no compression) is used automatically for files that wouldn't benefit from compression — typically already-compressed formats like `.jpg`, `.mp4`, or nested `.zip` files — since compressing already-compressed data wastes CPU for negligible or even negative size benefit.

---

**Q8 🔥 What is Zip64, and why does it exist?**
> The original 1989 ZIP specification uses 32-bit fields, capping individual files at 4 GB, archives at roughly 4 GB total, and 65,535 entries per archive. Zip64 is an extension using 64-bit fields to remove these limits, applied automatically by modern zip/unzip tools when an archive or file exceeds the original limits. Older or minimal ZIP implementations that don't understand Zip64 can fail or truncate when reading such an archive.

---

**Q9. Why can a large zip file sometimes report "End of central directory signature not found" even though most of the file downloaded successfully?**
> Standard unzip implementations locate the Central Directory by seeking to the **end** of the file first. If the download was interrupted or the file was truncated, that end-of-file structure is missing or incomplete, and unzip can't build its index of entries — even if the earlier portion of the file (containing actual file data) downloaded completely intact. This is a structural consequence of where ZIP stores its index, unlike a sequential format like tar.gz.

---

## zip vs Other Formats

**Q10 🔥 Why would you choose .zip over .tar.gz for sharing a file with someone whose operating system you don't know?**
> ZIP has truly universal native support — Windows Explorer, macOS Finder, and virtually every Linux desktop environment can open a `.zip` with no additional software installed. `.tar.gz` is native on Linux/macOS but typically requires installing extra software on Windows to open conveniently, making ZIP the safer default when you can't assume the recipient's technical environment or tooling.

---

**Q11. Why might a Unix sysadmin prefer tar (with gzip) over zip for backing up a server's files?**
> `tar` was designed for Unix from the start and preserves file permissions, ownership, and symlinks with full fidelity as a core part of the format. ZIP's Unix permission support is a later, inconsistently-implemented extension ("extra field" data), so exact permission bits and ownership information can be lost or altered depending on which zip/unzip implementation created and later read the archive — a real concern for backups meant to be restored exactly as they were.

---

**Q12 🔥 When would you choose 7z over both zip and tar.gz?**
> When maximum compression ratio matters more than universal compatibility — 7z's LZMA2 algorithm typically compresses noticeably smaller than either Deflate (zip) or gzip (tar.gz). It's also the better choice when strong built-in encryption (AES-256 by default) is needed, since ZIP's traditional password protection is cryptographically weak by comparison.

---

## Compression & Options

**Q13. What does `zip -0` do, and when is it useful?**
> `-0` stores files with **no compression at all** ("stored" method), making the operation much faster since there's no compression work to do. It's most useful for bundling files that are already compressed (video, images, other archives) where further compression would waste CPU time for negligible or no size reduction — you're using zip purely as a container/bundler in that case.

---

**Q14 🔥 What's the difference between `zip -u` (update) and `zip -f` (freshen)?**
> `-u` (update) adds files that don't yet exist in the archive AND replaces entries whose source file is newer than what's stored. `-f` (freshen) only ever replaces **existing** entries with newer versions — it never adds a file to the archive that wasn't already present in it.

---

**Q15. Why does `zip -y` matter when archiving a directory containing symbolic links?**
> Without `-y`, zip may **follow** the symlink and store a copy of the target file's actual content under the symlink's name — silently changing the semantics of what gets archived. With `-y`, zip stores the symlink itself (its target path) as a symlink entry, so extracting the archive elsewhere correctly recreates a symlink pointing to the same path, matching how `tar` handles symlinks by default.

---

## Security

**Q16 🔥 Is the password set with `zip -e` or `zip -P` cryptographically strong?**
> No. Standard Info-ZIP's default encryption is **ZipCrypto**, a stream cipher from 1989 with well-documented, practical known-plaintext attacks — tools exist that can recover the password in a short time if any predictable content exists inside the archive (which is common, since many file formats have recognizable headers). It should not be relied on to protect genuinely sensitive data; layering GPG encryption on top of the archive, or using a format with strong AES encryption by default (like 7z), is the safer approach.

---

**Q17. Why is `zip -P "password" archive.zip file.txt` considered a bad practice compared to `zip -e`?**
> Supplying the password directly on the command line with `-P` exposes it in **two** persistent places: the shell's command history file, and the process list (visible to any other user on the same system via `ps aux` while the command runs). `-e` prompts interactively instead, so the password is typed directly and never appears in either location.

---

**Q18 🔥 What is "Zip Slip," and why is it dangerous?**
> Zip Slip is a path traversal vulnerability where a maliciously crafted archive contains an entry with a relative path like `../../../etc/cron.d/malicious`. If the extracting tool doesn't validate that each entry's resolved path stays within the intended destination directory, extraction can write files **outside** that directory entirely — potentially overwriting system files or planting malicious files in sensitive locations. Modern command-line `unzip` has mitigations and typically warns or refuses on such entries, but not every library or application that handles ZIP files implements this protection correctly, so extracting untrusted archives (especially via custom application code) always carries this risk.

---

**Q19. Why should you check `unzip -l` before extracting an archive from an untrusted source?**
> `unzip -l` shows both compressed and uncompressed size per entry without actually extracting anything, letting you spot a **zip bomb** — an archive with a wildly disproportionate compression ratio (e.g., a few KB that would expand into many gigabytes) — before committing disk space or triggering a denial-of-service condition by extracting it blindly.

---

## Scenario-Based

**Q20 🔥 A cron job runs `unzip archive.zip -d /opt/app` nightly, but it started hanging indefinitely after a deployment left old files in place. What happened, and how do you fix it?**
> Without `-o` or `-n`, `unzip` prompts interactively whenever it encounters a file that already exists at the destination — asking "replace existing_file? [y]es/[n]o/[A]ll/[N]one/[r]ename." In a cron job with no attached terminal, that prompt has no way to be answered and the process simply hangs forever, silently consuming a cron slot. Fix: always be explicit about overwrite behavior in non-interactive contexts — `unzip -o archive.zip -d /opt/app` to force overwrite, or `unzip -n` if existing files should be preserved untouched.

---

**Q21. You extract a zip archive that was created on Windows, and Unix filenames with accented characters (like "résumé.docx") come out garbled. Why, and how do you fix it?**
> This is a filename encoding mismatch — older ZIP tools may store filenames using a legacy codepage (like CP437) rather than UTF-8, and if the archive doesn't correctly signal UTF-8 encoding via the general purpose bit flag, the extracting tool may misinterpret the byte sequence, producing "mojibake" garbled characters. Fix: try forcing UTF-8 interpretation during extraction (`unzip -O UTF-8 archive.zip`), and ensure the creating tool uses a modern Info-ZIP build that defaults to UTF-8 filename encoding.

---

**Q22 🔥 You archive a shell script with `chmod 700` set, but after zipping and unzipping it (possibly on a different machine), it comes back with different permissions. Why, and what's a more reliable approach if exact permissions matter?**
> ZIP's support for Unix permission metadata is an "extra field" extension bolted onto a format originally designed for DOS — it's not guaranteed to be written or interpreted consistently across every zip/unzip implementation, so permission bits can be lost, reset to a default, or altered in transit. If preserving exact Unix permissions and ownership is important (e.g., server backups, deployment archives), `tar` (designed natively for Unix metadata) is the more reliable choice; when zip must be used for compatibility reasons, always explicitly re-apply critical permissions (`chmod`) after extraction rather than trusting the archive to have preserved them.

---

**Q23. A colleague wants to exclude all `.log` files anywhere in a project tree using `zip -r archive.zip project/ -x *.log`, but some log files still end up in the archive. What's likely wrong?**
> The exclude pattern almost certainly wasn't quoted. Without quotes, the **shell** expands `*.log` into a literal list of matching filenames in the *current* directory before zip ever receives the argument — so zip ends up excluding only those specific files (if any matched at all), not applying the pattern recursively throughout the tree as intended. The fix is to quote the pattern so zip receives and interprets the raw glob itself: `zip -r archive.zip project/ -x "*.log"`.

---

**Q24. Why might `unzip -l suspicious.zip` be a critical safety step before running `unzip suspicious.zip` on an archive received from an unknown source?**
> It reveals each entry's compressed and uncompressed size without extracting any data, allowing a quick sanity check for a zip bomb — a maliciously small file that would decompress into an enormous amount of data, potentially filling available disk space or causing a denial-of-service condition, especially dangerous in automated pipelines that extract archives without human review.

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md)
