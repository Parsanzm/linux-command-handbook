# tar — The Complete Reference

> Archive files and directories on Unix-like systems

`tar` is one of the oldest and most important Unix utilities. Although it was originally designed for writing files to magnetic tapes, today it is primarily used for archiving directories, creating backups, packaging source code, and transferring large collections of files.

Unlike utilities such as `zip`, `tar` itself **does not compress data**. Instead, it combines multiple files into a single archive and optionally passes the archive through a compression program such as `gzip`, `bzip2`, `xz`, or `zstd`.

Whether you are a Linux beginner, system administrator, DevOps engineer, security researcher, or software developer, understanding `tar` is essential.

---

# Table of Contents

- [What is tar?](#what-is-tar)
- [History](#history)
- [Where does tar live?](#where-does-tar-live)
- [How tar works internally](#how-tar-works-internally)
- [Archive Format](#archive-format)
- [Tar Header Layout](#tar-header-layout)
- [512-byte Block Structure](#512-byte-block-structure)
- [Syntax](#syntax)
- [Operation Modes](#operation-modes)
- [Compression Methods](#compression-methods)
- [All Options](#all-options)
- [Creating Archives](#creating-archives)
- [Extracting Archives](#extracting-archives)
- [Listing Archives](#listing-archives)
- [Appending Files](#appending-files)
- [Updating Archives](#updating-archives)
- [Deleting Files](#deleting-files)
- [Incremental Backups](#incremental-backups)
- [Metadata Preservation](#metadata-preservation)
- [Permissions and Ownership](#permissions-and-ownership)
- [ACLs, SELinux, and xattrs](#acls-selinux-and-xattrs)
- [Sparse Files](#sparse-files)
- [Hard Links](#hard-links)
- [Compression Comparison](#compression-comparison)
- [Performance](#performance)
- [Security Considerations](#security-considerations)
- [GNU vs BSD tar](#gnu-vs-bsd-tar)
- [Environment Variables](#environment-variables)
- [Exit Codes](#exit-codes)
- [Alternatives](#alternatives)
- [Related Commands](#related-commands)

---

# What is tar?

`tar` stands for **Tape Archive**.

Originally, it was created to write files sequentially onto magnetic tapes.

Modern operating systems rarely use tape drives, but the archive format has remained the de facto standard for packaging files on Unix and Linux.

Today, `tar` is commonly used for:

- Creating backups
- Packaging source code
- Software distribution
- Docker image exports
- System migration
- Log archival
- Offline storage
- Data transfer
- Combining thousands of files into one archive

Unlike ZIP, TAR separates two independent tasks:

1. Archiving
2. Compression

This design allows any compression algorithm to be used without changing the archive format.

For example:

```bash
tar -cf archive.tar project/
```

creates an archive only.

While

```bash
tar -czf archive.tar.gz project/
```

creates an archive and compresses it using gzip.

Likewise,

```bash
tar -cJf archive.tar.xz project/
```

uses xz,

and

```bash
tar --zstd -cf archive.tar.zst project/
```

uses Zstandard.

---

# History

The first version of tar appeared in Version 7 Unix around 1979.

Its primary purpose was storing files onto magnetic tape devices.

The name comes directly from:

```
Tape ARchive
```

As Unix evolved, the tar format became standardized.

Several formats now exist.

| Format | Description |
|---------|-------------|
| V7 | Original Unix format |
| USTAR | POSIX standard |
| GNU tar | GNU extensions |
| PAX | POSIX.1-2001 extended format |

Modern GNU tar automatically chooses an appropriate format unless explicitly specified.

---

# Where does tar live?

Most Linux systems install tar in one of the following locations.

```
/bin/tar
/usr/bin/tar
```

Verify on your machine:

```bash
which tar
```

```
/usr/bin/tar
```

Show every executable found:

```bash
type -a tar
```

Check version:

```bash
tar --version
```

Example:

```
tar (GNU tar) 1.35
```

Determine whether tar is aliased:

```bash
type tar
```

Find executable without aliases:

```bash
command -v tar
```

---

# How tar works internally

Internally, tar performs a sequential archive operation.

```
Read filesystem

↓

stat()

↓

Collect metadata

↓

Generate TAR headers

↓

Read file contents

↓

Write archive sequentially

↓

(Optional)

Pipe through compressor

↓

Write final archive
```

Unlike ZIP, tar never seeks backward while creating archives.

Instead, it writes records sequentially.

This makes tar extremely efficient for streaming data.

For example:

```
Filesystem

↓

tar

↓

stdout

↓

gzip

↓

ssh

↓

remote host
```

This design allows archives to be created without temporary files.

Example:

```bash
tar -cf - project | gzip | ssh server "cat > backup.tar.gz"
```

No intermediate archive ever exists on disk.

This streaming architecture is one reason tar remains popular after more than four decades.

---

# Archive Format

A TAR archive consists of a sequence of records.

Every file contributes:

- one header
- file contents
- padding

```
+------------------+
| Header           |
+------------------+
| File Data        |
+------------------+
| Padding          |
+------------------+

↓

Header

↓

File

↓

Header

↓

File

↓

Header

↓

File

↓

End of archive
```

Unlike ZIP, no central directory exists.

Reading starts from the beginning.

Consequently,

Listing the final file often requires scanning the entire archive.

Appending new files is simple because new entries are written to the end.

Deleting entries, however, requires rebuilding the archive.

---
