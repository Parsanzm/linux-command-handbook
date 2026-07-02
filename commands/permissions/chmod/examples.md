# chmod — Practical Examples

> Real-world patterns from web servers, scripting, DevOps, and everyday sysadmin work.

---

## Table of Contents

- [Basic Octal Usage](#basic-octal-usage)
- [Basic Symbolic Usage](#basic-symbolic-usage)
- [Making Scripts Executable](#making-scripts-executable)
- [Recursive Changes](#recursive-changes)
- [Web Server Permissions](#web-server-permissions)
- [SSH Keys & Sensitive Files](#ssh-keys--sensitive-files)
- [Special Permissions in Practice](#special-permissions-in-practice)
- [Copying Permissions Between Files](#copying-permissions-between-files)
- [Combining chmod with find](#combining-chmod-with-find)
- [Verbose & Reporting](#verbose--reporting)
- [Real-World Recipes](#real-world-recipes)

---

## Basic Octal Usage

```bash
# Owner: read/write/execute, group & others: read/execute
chmod 755 script.sh

# Owner: read/write, group & others: read-only
chmod 644 config.yaml

# Private, owner only
chmod 700 ~/bin
chmod 600 ~/.ssh/id_rsa

# Fully open (avoid unless truly needed)
chmod 777 shared_tmp/

# No access to anyone but root
chmod 000 locked_file
```

---

## Basic Symbolic Usage

```bash
# Add execute permission for the owner
chmod u+x deploy.sh

# Remove write permission for group and others
chmod go-w secrets.conf

# Set exact permissions for others
chmod o=r public_notes.txt

# Add read for everyone
chmod a+r README.md

# Multiple classes, comma-separated (no spaces around commas)
chmod u=rwx,g=rx,o= private_script.sh

# Toggle execute for owner+group at once
chmod ug+x run.sh
```

---

## Making Scripts Executable

```bash
# Standard: make a script runnable by its owner
chmod +x deploy.sh
./deploy.sh

# Make executable for everyone (e.g., shared CLI tool)
chmod a+x /usr/local/bin/mytool

# Make an entire folder of scripts executable
chmod +x scripts/*.sh

# Common pattern after git clone / cloning a repo with scripts
git clone https://example.com/repo.git
cd repo
chmod +x install.sh
./install.sh
```

---

## Recursive Changes

```bash
# Recursively set all files & dirs to 755 (dangerous for files, see edge-cases)
chmod -R 755 project/

# Better: separate rules for files vs directories
find project/ -type d -exec chmod 755 {} \;   # directories: 755
find project/ -type f -exec chmod 644 {} \;   # files: 644

# Using capital X: directories get +x, files keep their existing x state
chmod -R a+rX project/
# Directories become traversable
# Already-executable files stay executable
# Plain data files (images, text, etc.) do NOT become executable

# Recursive verbose (see every file changed)
chmod -Rv 644 configs/
```

---

## Web Server Permissions

```bash
# Typical Apache/Nginx document root setup
sudo chown -R www-data:www-data /var/www/html
sudo find /var/www/html -type d -exec chmod 755 {} \;
sudo find /var/www/html -type f -exec chmod 644 {} \;

# Writable uploads directory for the web server user only
sudo chmod 750 /var/www/html/uploads
sudo chown www-data:www-data /var/www/html/uploads

# WordPress-style hardened permissions
find /var/www/wordpress -type d -exec chmod 755 {} \;
find /var/www/wordpress -type f -exec chmod 644 {} \;
chmod 600 /var/www/wordpress/wp-config.php   # sensitive credentials file

# Shared group-writable directory for a deploy team
sudo chgrp -R deploy /var/www/app
sudo chmod -R 2775 /var/www/app     # setgid so new files inherit "deploy" group
```

---

## SSH Keys & Sensitive Files

```bash
# SSH requires strict permissions or it refuses to use the key
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_rsa
chmod 644 ~/.ssh/id_rsa.pub
chmod 600 ~/.ssh/authorized_keys
chmod 600 ~/.ssh/config

# Without correct perms, SSH errors out:
# "Permissions 0644 for 'id_rsa' are too open"

# GPG directory
chmod 700 ~/.gnupg

# Environment / secrets files
chmod 600 .env
chmod 600 credentials.json
```

---

## Special Permissions in Practice

```bash
# setuid: allow a program to run with owner's privileges
sudo chmod u+s /usr/local/bin/special_tool
sudo chmod 4755 /usr/local/bin/special_tool

# setgid on a shared team directory — new files inherit the group
sudo chmod g+s /srv/shared/team
sudo chmod 2775 /srv/shared/team
sudo chgrp team /srv/shared/team

# sticky bit on a shared upload folder — users can't delete each other's files
sudo chmod +t /srv/shared/uploads
sudo chmod 1777 /srv/shared/uploads

# Combine setgid + sticky + rwx for a fully shared collaborative folder
sudo chmod 3775 /srv/shared/collab   # setgid(2000) + sticky(1000) = 3000 + 775
```

---

## Copying Permissions Between Files

```bash
# Copy exact mode from one file to another
chmod --reference=template.txt target.txt

# Apply a reference file's permissions to many files
chmod --reference=template.conf *.conf

# Useful when replacing a config file but wanting to keep prior permissions
cp new_config.conf /etc/app/config.conf.new
chmod --reference=/etc/app/config.conf /etc/app/config.conf.new
mv /etc/app/config.conf.new /etc/app/config.conf
```

---

## Combining chmod with find

```bash
# Fix all directories to 755, leave files alone
find . -type d -exec chmod 755 {} +

# Fix all files to 644, leave directories alone
find . -type f -exec chmod 644 {} +

# Make only .sh files executable
find . -name "*.sh" -exec chmod +x {} +

# Remove execute permission from all files in a directory (but not dirs)
find /var/www -type f -exec chmod -x {} +

# Find and fix world-writable files (security audit)
find / -xdev -type f -perm -002 -exec chmod o-w {} +

# Find files with insecure permissions before fixing
find /etc -type f -perm /o+w
```

---

## Verbose & Reporting

```bash
# See exactly what changed, one line per file
chmod -c 644 *.txt
# mode of 'notes.txt' changed from 0755 (rwxr-xr-x) to 0644 (rw-r--r--)

# See every file processed, even unchanged ones
chmod -v 644 *.txt
# mode of 'already_644.txt' retained as 0644 (rw-r--r--)

# Dry-run style check before applying (no native --dry-run; use find -perm to preview)
find . -type f ! -perm 644 -print   # show files that would change
```

---

## Real-World Recipes

```bash
# --- Deployment ---

# Fresh deploy: fix permissions after extracting an archive
tar -xzf release.tar.gz -C /opt/app/
find /opt/app -type d -exec chmod 755 {} +
find /opt/app -type f -exec chmod 644 {} +
chmod +x /opt/app/bin/*

# --- Security Hardening ---

# Lock down a directory of secrets
chmod 700 /etc/app/secrets
find /etc/app/secrets -type f -exec chmod 600 {} +

# Remove world-write everywhere under a path (common audit fix)
find /var/www -type f -perm -o+w -exec chmod o-w {} +
find /var/www -type d -perm -o+w -exec chmod o-w {} +

# --- Docker / Containers ---

# Ensure entrypoint scripts are executable inside an image build context
chmod +x docker-entrypoint.sh

# --- Cron Jobs & Automation ---

# Make a script executable and owned correctly for a cron job
chmod 750 /opt/scripts/nightly_backup.sh
chown root:cron /opt/scripts/nightly_backup.sh

# --- Shared Team Environments ---

# Set up a collaborative directory: group-writable, new files inherit group
mkdir -p /srv/shared/team_project
chgrp team_project /srv/shared/team_project
chmod 2775 /srv/shared/team_project
```

---

> See also: [`README.md`](README.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
