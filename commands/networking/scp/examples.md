# scp — Practical Examples

> Real-world patterns for copying files to, from, and between remote
> hosts securely.

---

## Table of Contents

- [Basic Usage](#basic-usage)
- [Copying Directories](#copying-directories)
- [Using a Specific Key or Port](#using-a-specific-key-or-port)
- [Remote-to-Remote Copies](#remote-to-remote-copies)
- [Preserving Attributes and Limiting Bandwidth](#preserving-attributes-and-limiting-bandwidth)
- [Combining scp with Other Tools](#combining-scp-with-other-tools)
- [Real-World Recipes](#real-world-recipes)

---

## Basic Usage

```bash
# Copy a local file to a remote server
scp report.pdf alice@server.example.com:/home/alice/reports/

# Copy a remote file to the local machine
scp alice@server.example.com:/home/alice/report.pdf ./

# Copy to the current directory, keeping the same filename
scp alice@server.example.com:/home/alice/report.pdf .

# Copy multiple files in a single command
scp file1.txt file2.txt file3.txt alice@server.example.com:/tmp/
```

---

## Copying Directories

```bash
# Recursively copy an entire directory to a remote host
scp -r ./project/ alice@server.example.com:/home/alice/project/

# Recursively copy a remote directory back to the local machine
scp -r alice@server.example.com:/home/alice/project/ ./project/

# Recursive copy while preserving timestamps and permissions
scp -rp ./project/ alice@server.example.com:/home/alice/project/
```

---

## Using a Specific Key or Port

```bash
# Use a specific private key
scp -i ~/.ssh/id_ed25519_work file.txt alice@server.example.com:/tmp/

# Connect on a non-default SSH port (capital -P, not lowercase)
scp -P 2222 file.txt alice@server.example.com:/tmp/

# Combine both
scp -P 2222 -i ~/.ssh/id_ed25519_work file.txt alice@server.example.com:/tmp/
```

---

## Remote-to-Remote Copies

```bash
# Copy directly between two remote hosts (data still typically
# routes through your local machine unless -3 controls this explicitly)
scp alice@server1.example.com:/data/export.csv bob@server2.example.com:/import/

# Explicitly route through the local machine for a remote-to-remote copy
scp -3 alice@server1.example.com:/data/export.csv bob@server2.example.com:/import/
```

---

## Preserving Attributes and Limiting Bandwidth

```bash
# Preserve modification times and file permissions
scp -p config.yaml alice@server.example.com:/etc/myapp/

# Limit transfer speed to avoid saturating a shared connection
scp -l 1000 largefile.iso alice@server.example.com:/tmp/
# -l is in Kbit/s, so 1000 = roughly 125 KB/s

# Compress data during transfer, useful over slow links for
# already-compressible content
scp -C large_text_dump.sql alice@server.example.com:/tmp/
```

---

## Combining scp with Other Tools

```bash
# Copy a file, then immediately verify its checksum matches
scp release.tar.gz alice@server.example.com:/tmp/
ssh alice@server.example.com "sha256sum /tmp/release.tar.gz"
sha256sum release.tar.gz
# compare the two outputs manually

# Archive a directory locally, then transfer just the single archive
tar czf project.tar.gz ./project/
scp project.tar.gz alice@server.example.com:/home/alice/
ssh alice@server.example.com "cd /home/alice && tar xzf project.tar.gz"

# Pipe a local tar stream directly through ssh without an intermediate
# archive file at all — often faster for many small files
tar czf - ./project/ | ssh alice@server.example.com "tar xzf - -C /home/alice/"
```

---

## Real-World Recipes

```bash
# --- Deploy a Build Artifact to a Server ---
scp -C dist/app.tar.gz deploy@server.example.com:/tmp/
ssh deploy@server.example.com "cd /opt/app && tar xzf /tmp/app.tar.gz && systemctl restart myapp"

# --- Pull the Latest Log File for Local Inspection ---
scp alice@server.example.com:/var/log/myapp/latest.log ./

# --- Back Up a Remote Config Directory Before Making Changes ---
scp -rp alice@server.example.com:/etc/myapp/ ./config-backup-$(date +%F)/

# --- Copy a File to Several Servers in a Loop ---
for server in web1 web2 web3; do
  scp deploy.conf alice@"$server":/etc/myapp/deploy.conf
done

# --- Transfer a Large Dataset with a Bandwidth Cap During Business Hours ---
scp -l 2000 dataset.tar.gz alice@server.example.com:/data/

# --- Copy an SSH Public Key Manually, Without ssh-copy-id ---
scp ~/.ssh/id_ed25519.pub alice@server.example.com:/tmp/
ssh alice@server.example.com "cat /tmp/id_ed25519.pub >> ~/.ssh/authorized_keys && rm /tmp/id_ed25519.pub"
```

---

> See also: [`README.md`](README.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
