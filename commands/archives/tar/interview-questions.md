# tar — Interview Questions

> Covers beginner to senior level. Each question includes the answer.
> Questions marked 🔥 are commonly asked in real interviews.

---

## Table of Contents

- [Conceptual](#conceptual)
- [Basic Usage](#basic-usage)
- [Compression](#compression)
- [Extraction & Paths](#extraction--paths)
- [Permissions & Security](#permissions--security)
- [Backup & Advanced](#backup--advanced)
- [Scenario-Based](#scenario-based)
- [GNU vs BSD](#gnu-vs-bsd)

---

## Conceptual

**Q1. What does tar stand for and what was it originally designed for?**
> **Tape ARchive**. Designed in 1979 to write files sequentially to magnetic tape drives for backup. The format remains sequential today — no central index — which is why extracting specific files from a large archive requires reading from the beginning.

---

**Q2 🔥 Does tar compress files? Explain.**
> No — `tar` only **bundles** files into a single stream (archive). Compression is handled by a separate tool: `gzip`, `bzip2`, `xz`, or `zstd`. They can be combined:
> - via pipe: `tar -c dir/ | gzip > archive.tar.gz`
> - via flag: `tar -czf archive.tar.gz dir/` (the `-z` flag tells tar to pipe through gzip internally)
>
> This separation follows the Unix philosophy: each tool does one thing well.

---

**Q3 🔥 What is the structure of a tar archive internally?**
> A tar archive is a sequential stream of **512-byte blocks**. Each file consists of:
> 1. A **512-byte header block**: filename (100 bytes in USTAR), permissions, UID/GID, file size, mtime, checksum, file type, link name
> 2. **File data** padded to a 512-byte boundary
>
> The archive ends with **two consecutive 512-byte zero blocks**. There is no central directory or index — unlike ZIP.

---

**Q4. What are the three main tar operations?**
> - `-c` / `--create` — create a new archive
> - `-x` / `--extract` — extract files from an archive
> - `-t` / `--list` — list contents without extracting
>
> Plus less common: `-r` (append), `-u` (update), `-d` (compare/diff).

---

**Q5. What is the difference between `.tar.gz`, `.tgz`, `.tar.bz2`, `.tar.xz`?**
> All are tar archives with different compression:
> - `.tar.gz` / `.tgz` — gzip compression (fast, widely supported)
> - `.tar.bz2` / `.tbz2` — bzip2 compression (better ratio, slower)
> - `.tar.xz` / `.txz` — xz/LZMA compression (best ratio, slowest)
> - `.tar.zst` — zstandard (near-xz ratio at gzip speed — modern choice)
>
> The compression format doesn't change how tar works — only how the byte stream is encoded.

---

## Basic Usage

**Q6 🔥 What does `tar -czf archive.tar.gz dir/` do? Explain each flag.**
> - `-c` — create a new archive
> - `-z` — compress with gzip
> - `-f archive.tar.gz` — write to this file (must be the last flag before the filename)
> - `dir/` — the directory to archive
>
> Old-style equivalent: `tar czf archive.tar.gz dir/` (no dash).

---

**Q7 🔥 What is the difference between `tar -cvf` and `tar -czvf`?**
> Both create an archive with verbose output. `-z` adds gzip compression. Without `-z`, the result is an uncompressed `.tar` file. With `-z`, it's a `.tar.gz` file.

---

**Q8. How do you list the contents of a tar archive without extracting?**
> ```bash
> tar -tf archive.tar.gz       # list filenames
> tar -tvf archive.tar.gz      # list with details (like ls -l)
> ```

---

**Q9 🔥 How do you extract a tar archive to a specific directory?**
> ```bash
> tar -xzf archive.tar.gz -C /target/directory/
> # -C changes to the directory before extracting
> # The directory must already exist
> mkdir -p /target/directory/ && tar -xzf archive.tar.gz -C /target/directory/
> ```

---

**Q10. How do you extract only specific files from an archive?**
> ```bash
> tar -xvf archive.tar.gz specific_file.txt
> tar -xvf archive.tar.gz dir/subdir/file.txt
> tar -xvf archive.tar.gz --wildcards '*.conf'
> ```

---

**Q11. What does `--strip-components=N` do?**
> Strips N leading path components from filenames during extraction.
> ```bash
> # Archive contains: project/src/main.c
> tar -xf archive.tar --strip-components=1
> # Extracts as: src/main.c  (stripped "project/")
>
> tar -xf archive.tar --strip-components=2
> # Extracts as: main.c  (stripped "project/src/")
> ```
> Commonly used when extracting software tarballs that have a top-level directory you want to skip.

---

## Compression

**Q12 🔥 Compare gzip, bzip2, xz, and zstd for use with tar.**
> | Format | Flag | Speed | Ratio | Best for |
> |--------|------|-------|-------|---------|
> | gzip | `-z` | Fast | Good | General use, wide compatibility |
> | bzip2 | `-j` | Slow | Better | When ratio matters more than speed |
> | xz | `-J` | Slowest | Best | Distribution packages, maximum compression |
> | zstd | `--zstd` | Fast | Very good | Modern systems, best speed/ratio balance |
>
> For most purposes: gzip for speed, xz for minimum size, zstd for modern pipelines.

---

**Q13. How do you use multi-threaded compression with tar?**
> ```bash
> # pigz: parallel gzip (drop-in replacement)
> tar -c dir/ | pigz -p 8 > archive.tar.gz
>
> # pbzip2: parallel bzip2
> tar -c dir/ | pbzip2 -p8 > archive.tar.bz2
>
> # xz with threads
> tar -c dir/ | xz -T 8 > archive.tar.xz
>
> # zstd with threads
> tar -c dir/ | zstd -T8 > archive.tar.zst
> ```

---

**Q14. What does `tar -caf archive.tar.xz dir/` do?**
> The `-a` flag tells tar to **auto-detect the compression format from the file extension**. Since the extension is `.xz`, tar automatically uses xz compression. Equivalent to `tar -cJf archive.tar.xz dir/`.

---

## Extraction & Paths

**Q15 🔥 Why does GNU tar print "Removing leading '/' from member names"?**
> When archiving absolute paths (e.g., `tar -czf backup.tar.gz /etc/`), tar stores paths like `/etc/nginx/nginx.conf`. On extraction, this would overwrite `/etc/nginx/nginx.conf` on the target system — a security risk. GNU tar removes the leading `/` by default, making paths relative, and warns you. Use `-P` / `--absolute-names` to override (dangerous with untrusted archives).

---

**Q16 🔥 What security risk exists when extracting untrusted tar archives?**
> Two main risks:
> 1. **Absolute paths**: archive contains `/etc/passwd` which overwrites the real `/etc/passwd` (GNU tar strips leading `/` by default, but `-P` disables this).
> 2. **Path traversal**: archive contains `../../etc/cron.d/backdoor` which escapes the extraction directory.
>
> Safe practice: always inspect before extracting:
> ```bash
> tar -tf untrusted.tar | grep -E "^/|^\.\."
> # Extract to isolated directory:
> mkdir /tmp/safe && tar -xf untrusted.tar -C /tmp/safe
> ```

---

**Q17. How do you prevent tar from overwriting existing files during extraction?**
> ```bash
> tar -xf archive.tar -k              # --keep-old-files: skip if exists
> tar -xf archive.tar --keep-newer-files  # skip if newer on disk
> ```

---

**Q18. What happens when you run `tar -xf archive.tar` without `-C`?**
> Files are extracted relative to the **current directory**. If the archive contains `dir/file.txt`, it's extracted to `./dir/file.txt`. This can be dangerous if you're in an important directory like `/` or `/etc`.

---

## Permissions & Security

**Q19 🔥 Does extracting a tar archive as a non-root user restore file ownership?**
> No. Only root can change file ownership (`chown`). A non-root user extracting an archive gets files owned by themselves, regardless of what ownership is stored in the archive.
>
> To restore ownership: `sudo tar -xpf archive.tar.gz -C /restore/`

---

**Q20. What does `-p` do in `tar -xpf`?**
> `-p` / `--preserve-permissions` restores exact permissions from the archive, ignoring the current umask. Without `-p`, extracted file permissions are modified by the umask (e.g., 755 becomes 644 with umask 022). For root, `-p` is the default.

---

**Q21. How does tar handle SUID/SGID bits during extraction for non-root users?**
> GNU tar strips SUID and SGID bits when extracting as non-root — intentional security behavior. Even with `-p`, a non-root user can't create SUID files. Root using `sudo tar -xpf` restores SUID/SGID bits.

---

## Backup & Advanced

**Q22 🔥 How do you create an incremental backup with tar?**
> ```bash
> # Full backup (day 1):
> tar -czf full.tar.gz --listed-incremental=snapshot.snar /data/
>
> # Incremental (day 2+, only changed files):
> tar -czf incr_day2.tar.gz --listed-incremental=snapshot.snar /data/
>
> # Restore: apply full then each incremental in order
> tar -xzf full.tar.gz --listed-incremental=/dev/null -C /restore/
> tar -xzf incr_day2.tar.gz --listed-incremental=/dev/null -C /restore/
> ```
> The `snapshot.snar` file records file modification times. Keep it alongside the archives.

---

**Q23. How do you use tar to copy a directory tree efficiently?**
> ```bash
> tar -cf - source/ | tar -xf - -C /destination/
> # Faster than cp -r for many small files (one pass, no filesystem overhead)
> # Also preserves permissions and timestamps
> ```

---

**Q24. How do you create a tar archive from a list of files?**
> ```bash
> # From a file:
> tar -czf archive.tar.gz -T filelist.txt
>
> # From find output (null-safe for special filenames):
> find . -name "*.py" -print0 | tar -czf archive.tar.gz --null -T -
> ```

---

**Q25 🔥 How do you handle sparse files in tar?**
> Sparse files have "holes" — regions of zeros that aren't physically stored on disk. Without `--sparse`, tar archives all the zeros, inflating the archive massively.
> ```bash
> tar -S -czf archive.tar.gz sparse.img    # -S = --sparse
> tar --sparse -czf archive.tar.gz sparse.img
> ```
> Common examples of sparse files: VM disk images, database files, pre-allocated log files.

---

**Q26. How do you verify a tar archive after creation?**
> ```bash
> # Test integrity (decompress and read all blocks):
> tar -tzf archive.tar.gz > /dev/null && echo "OK"
> gzip -t archive.tar.gz && echo "OK"
>
> # Compare archive vs filesystem:
> tar -dvf archive.tar.gz
> # Shows files that differ between archive and disk
>
> # Full backup verification:
> tar -czf backup.tar.gz /data/ && \
>   tar -tzf backup.tar.gz > /dev/null && \
>   echo "Backup verified"
> ```

---

## Scenario-Based

**Q27 🔥 You need to deploy application files to a remote server without a temp file. How?**
> ```bash
> tar -czf - dist/ | ssh user@server "tar -xzf - -C /var/www/"
> # Creates archive on stdout, pipes over SSH, extracts on remote
> # No temporary file created on either end
> ```

---

**Q28. How do you download and extract a tarball in one command?**
> ```bash
> curl -sL https://example.com/app-1.0.tar.gz | tar -xzf -
> wget -qO- https://example.com/app-1.0.tar.gz | tar -xzf -
> # - means read from stdin
> ```

---

**Q29 🔥 A backup script runs nightly. Some files change during the backup and tar exits with code 1. Is the backup valid? What do you do?**
> Exit code 1 means "some files differed" — usually files that were being written while tar read them. The archive may contain inconsistent versions of those files.
>
> Options:
> 1. **Accept it**: if the files are logs (not critical state), code 1 is acceptable.
> 2. **Freeze the application**: pause writes before backup (e.g., `mysql flush tables with read lock`).
> 3. **Use snapshots**: LVM or ZFS snapshot → tar the snapshot (consistent state).
> 4. **Use application-aware tools**: `mysqldump`, `pg_dump` instead of file-level backup.
>
> In the backup script, distinguish exit codes:
> ```bash
> rc=$?; [ $rc -gt 1 ] && echo "FATAL" || [ $rc -eq 1 ] && echo "WARNING: files changed"
> ```

---

**Q30 🔥 What's wrong with this backup command?**
```bash
tar -czvf - /home/ > /backup/home.tar.gz
```
> **Two problems:**
> 1. `-v` (verbose) sends file list to **stdout**, which is also where the archive data goes (because `-f -` means stdout). Both the archive binary data and the verbose list get mixed into `/backup/home.tar.gz` — the archive will be corrupt.
> 2. Fix: send verbose to stderr:
> ```bash
> tar -czvf - /home/ 2>filelist.txt > /backup/home.tar.gz
> # or suppress verbose:
> tar -czf - /home/ > /backup/home.tar.gz
> ```

---

**Q31. How do you exclude `.git` and `node_modules` when creating a project archive?**
> ```bash
> tar -czf project.tar.gz project/ \
>   --exclude='.git' \
>   --exclude='node_modules' \
>   --exclude='__pycache__' \
>   --exclude='*.pyc'
>
> # Or use built-in VCS exclusion:
> tar --exclude-vcs -czf project.tar.gz project/
> ```

---

## GNU vs BSD

**Q32 🔥 How does GNU tar differ from BSD tar (macOS)?**
> Key differences:
> - GNU tar has `--exclude-vcs`, `--listed-incremental`, `--sparse`, `--acls` — BSD tar doesn't.
> - GNU tar uses `-J` for xz; BSD tar also supports this on newer versions.
> - GNU tar has full long option support; BSD tar is more limited.
> - On macOS, install `gnu-tar` via Homebrew (`gtar`) for GNU compatibility.
>
> For portable scripts, use short flags and avoid GNU-specific options, or detect the version:
> ```bash
> if tar --version 2>&1 | grep -q "GNU"; then
>   tar --exclude-vcs -czf archive.tar.gz dir/
> else
>   tar -czf archive.tar.gz dir/  # BSD: no --exclude-vcs
> fi
> ```

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md)
