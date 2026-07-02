# chown — Practical Examples

> Real-world patterns from web servers, backups, containers, and everyday system administration.

---

## Table of Contents

- [Basic Ownership Changes](#basic-ownership-changes)
- [Changing Owner and Group Together](#changing-owner-and-group-together)
- [Numeric UID/GID Usage](#numeric-uidgid-usage)
- [Recursive Ownership Changes](#recursive-ownership-changes)
- [Web Server & Application Deployment](#web-server--application-deployment)
- [Home Directory & User Migration](#home-directory--user-migration)
- [Targeted Changes with --from](#targeted-changes-with---from)
- [Copying Ownership Between Files](#copying-ownership-between-files)
- [Symlink-Specific Ownership](#symlink-specific-ownership)
- [Combining chown with find](#combining-chown-with-find)
- [Docker & Container Scenarios](#docker--container-scenarios)
- [Backup & Restore Scenarios](#backup--restore-scenarios)
- [Auditing Ownership](#auditing-ownership)
- [Real-World Recipes](#real-world-recipes)

---

## Basic Ownership Changes

```bash
# Change owner only
sudo chown alice document.txt

# Change group only (two equivalent forms)
sudo chown :developers document.txt
sudo chgrp developers document.txt

# Change owner, and set group to the owner's primary login group
sudo chown alice: document.txt

# Verify the change
ls -l document.txt
stat -c "%U:%G" document.txt
```

---

## Changing Owner and Group Together

```bash
# Most common real-world form: set both in one call
sudo chown alice:developers document.txt

# Multiple files at once
sudo chown alice:developers report.pdf notes.txt data.csv

# Using a glob
sudo chown alice:developers *.log

# Directory and everything inside it (see Recursive section for -R specifics)
sudo chown -R alice:developers /home/alice/project
```

---

## Numeric UID/GID Usage

```bash
# Using raw numbers instead of names — useful across systems with
# different /etc/passwd entries, or before a user account even exists locally
sudo chown 1000:1000 file.txt

# Mixed form: name for owner, number for group
sudo chown alice:1001 file.txt

# Look up a user's numeric IDs before using them
id alice
# uid=1000(alice) gid=1000(alice) groups=1000(alice),27(sudo)

# Look up what a numeric ID resolves to
getent passwd 1000
getent group 1000

# Common in disk-image/VM-migration workflows where numeric IDs are
# the only reliable link between two otherwise-unrelated systems
sudo chown -R 1000:1000 /mnt/restored_disk/home/originaluser
```

---

## Recursive Ownership Changes

```bash
# Take full ownership of a directory tree
sudo chown -R alice:alice /home/alice

# Recursive + verbose (see every single file/dir processed)
sudo chown -Rv www-data:www-data /var/www/html

# Recursive + only report files that ACTUALLY changed (quieter, useful in logs)
sudo chown -Rc deploy:deploy /opt/app

# Recursive but explicitly refusing to follow symlinks out of the tree (default -P)
sudo chown -RP alice:alice /home/alice
# This is already the default; shown here to be explicit in scripts/documentation

# Recursive chown after restoring a backup archive
sudo tar -xzf backup.tar.gz -C /restore/
sudo chown -R originaluser:originaluser /restore/home/originaluser
```

---

## Web Server & Application Deployment

```bash
# Standard Apache/Nginx web root ownership
sudo chown -R www-data:www-data /var/www/html

# WordPress-style setup: web server owns everything, but restrict permissions too
sudo chown -R www-data:www-data /var/www/wordpress
sudo find /var/www/wordpress -type d -exec chmod 755 {} \;
sudo find /var/www/wordpress -type f -exec chmod 644 {} \;

# Application deployed by a "deploy" user, but SERVED by "www-data"
sudo chown -R deploy:www-data /srv/app
sudo chmod -R 750 /srv/app
# deploy user can read/write, www-data (group) can read/execute, others: nothing

# PHP-FPM pool running as a dedicated user needing write access to one directory
sudo chown -R php-fpm:php-fpm /var/www/app/storage
sudo chmod -R 775 /var/www/app/storage

# Node.js app managed by systemd running as its own service account
sudo useradd -r -s /usr/sbin/nologin nodeapp
sudo chown -R nodeapp:nodeapp /opt/nodeapp
```

---

## Home Directory & User Migration

```bash
# Fix ownership after manually creating a home directory
sudo mkdir /home/newuser
sudo chown newuser:newuser /home/newuser

# Migrating a user's files after renaming their account
sudo usermod -l newname oldname            # rename the account
sudo usermod -d /home/newname -m newname   # move home directory
sudo chown -R newname:newname /home/newname

# Restoring a home directory from backup, when the UID might differ
# between the backup source and this machine
id restoreduser
# uid=1002(restoreduser) ...
sudo chown -R restoreduser:restoreduser /home/restoreduser

# When a user's numeric UID was reused by ANOTHER account (classic gotcha —
# see edge-cases.md), always verify by name AND number:
sudo chown -R restoreduser:restoreduser /home/restoreduser
stat -c "%u %U" /home/restoreduser | sort -u
```

---

## Targeted Changes with --from

```bash
# Only re-assign files currently owned by "olduser:olduser" — leaves
# any files already owned by someone else completely untouched
sudo chown -R --from=olduser:olduser newuser:newuser /shared/project

# Useful for departmental reorganizations: migrate ONE person's files
# out of a shared, mixed-ownership directory without disturbing anyone else's
sudo chown -R --from=bob:contractors alice:employees /shared/workspace

# Combine with verbose to see exactly what qualified
sudo chown -Rv --from=olduser:olduser newuser:newuser /shared/project
```

---

## Copying Ownership Between Files

```bash
# Copy owner AND group from a reference file
sudo chown --reference=template.conf target.conf

# Apply one file's ownership pattern to many files
sudo chown --reference=correct_permissions_example.txt *.txt

# Common pattern: preserve ownership when replacing a config file
sudo cp new_nginx.conf /etc/nginx/nginx.conf.new
sudo chown --reference=/etc/nginx/nginx.conf /etc/nginx/nginx.conf.new
sudo mv /etc/nginx/nginx.conf.new /etc/nginx/nginx.conf
```

---

## Symlink-Specific Ownership

```bash
# Default: chown affects the TARGET of a symlink, not the link itself
ln -s /data/realfile.txt link.txt
sudo chown alice link.txt
stat -c "%U" /data/realfile.txt
# alice   ← the actual target's owner changed

# Change the symlink's OWN metadata without touching the target
sudo chown -h alice link.txt
stat -c "%U" link.txt        # (on systems supporting lchown, shows the symlink's own owner)

# Recursive chown by default does NOT follow symlinks during traversal (safe default)
sudo chown -R alice:alice /home/alice
# A symlink inside /home/alice pointing elsewhere on the filesystem is left alone

# Explicitly follow symlinks during a recursive operation (rarely needed, use with care)
sudo chown -R -L alice:alice /home/alice
```

---

## Combining chown with find

```bash
# Change ownership of only files (not directories) under a path
sudo find /var/www -type f -exec chown www-data:www-data {} +

# Change ownership of only directories
sudo find /var/www -type d -exec chown www-data:www-data {} +

# Change ownership of files matching a specific pattern
sudo find /var/log -name "*.log" -exec chown syslog:adm {} +

# Fix files currently owned by a deleted/nonexistent UID
sudo find / -xdev -nouser -exec chown alice:alice {} +

# Find and fix files owned by the wrong user before a security audit
sudo find /srv/app -not -user deploy -exec chown deploy:deploy {} +
```

---

## Docker & Container Scenarios

```bash
# Fix ownership of bind-mounted files created inside a container as root
docker run -v $(pwd):/app myimage touch /app/newfile.txt
sudo chown $(id -u):$(id -g) newfile.txt

# Run the container as your own UID/GID from the start (avoids the problem entirely)
docker run --user $(id -u):$(id -g) -v $(pwd):/app myimage touch /app/newfile.txt

# Inside a Dockerfile: create a dedicated app user and own the app directory
# (typical multi-stage or production image pattern)
# RUN useradd -r -u 1001 appuser
# COPY --chown=appuser:appuser . /app
# USER appuser

# Fixing volume ownership after a container ran as root by mistake
sudo chown -R 1000:1000 ./volume_data
```

---

## Backup & Restore Scenarios

```bash
# Full backup of a directory, then restore with original ownership intact
sudo tar -czpf backup.tar.gz /srv/app     # -p preserves ownership info in the archive
sudo tar -xzpf backup.tar.gz -C /restore/ # restore, root required to restore ownership

# If restoring on a DIFFERENT machine where UIDs don't line up:
sudo tar -xzf backup.tar.gz -C /restore/
sudo chown -R appuser:appuser /restore/srv/app   # fix ownership to match THIS machine's accounts

# rsync preserving ownership across machines (needs sudo/root on both ends)
sudo rsync -avz --owner --group -e ssh /srv/app/ user@remote:/srv/app/
```

---

## Auditing Ownership

```bash
# List files not owned by a valid, existing user (orphaned UIDs)
sudo find / -xdev -nouser

# List files not owned by a valid, existing group (orphaned GIDs)
sudo find / -xdev -nogroup

# Find all files owned by a specific user (before deleting their account, for example)
sudo find / -xdev -user alice

# Count files owned by each user under a directory
sudo find /srv/app -printf '%u\n' | sort | uniq -c | sort -rn

# Show raw numeric ownership instead of resolved names (useful across NFS/containers)
ls -n file.txt
stat -c "%u %g %U %G" file.txt
```

---

## Real-World Recipes

```bash
# --- New Application Deployment ---

sudo useradd -r -s /usr/sbin/nologin myapp
sudo mkdir -p /opt/myapp
sudo chown -R myapp:myapp /opt/myapp
sudo chmod -R 750 /opt/myapp

# --- Fixing a Broken Deployment After Files Were Extracted as root ---

sudo tar -xzf release.tar.gz -C /opt/myapp
sudo chown -R myapp:myapp /opt/myapp

# --- Preparing a Shared Group Directory ---

sudo groupadd project_team
sudo mkdir -p /srv/shared/project
sudo chown root:project_team /srv/shared/project
sudo chmod 2775 /srv/shared/project    # setgid so new files inherit "project_team"

# --- Departing Employee Cleanup ---

sudo find /srv/shared -user departing_employee -exec chown new_owner:project_team {} +
sudo userdel -r departing_employee

# --- Migrating an Account to a New UID Scheme ---

sudo find / -xdev -user 1005 -exec chown 2001 {} +   # by old numeric UID
sudo usermod -u 2001 alice                             # then update the account itself

# --- Restoring Ownership After a Botched "chown -R" Mistake ---

# If you have a recent backup or snapshot, restore just the metadata:
sudo rsync -a --owner --group --dry-run /backup/srv/app/ /srv/app/   # preview first
sudo rsync -a --owner --group /backup/srv/app/ /srv/app/             # apply
```

---

> See also: [`README.md`](README.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
