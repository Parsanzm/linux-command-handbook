# useradd — Practical Examples

> Real-world patterns from sysadmin, DevOps, security hardening, and automation.

---

## Table of Contents

- [Basic User Creation](#basic-user-creation)
- [System Accounts](#system-accounts)
- [Custom UID & GID](#custom-uid--gid)
- [Groups & Permissions](#groups--permissions)
- [Home Directory Control](#home-directory-control)
- [Shell & GECOS](#shell--gecos)
- [Account Expiry & Aging](#account-expiry--aging)
- [Skeleton Directory](#skeleton-directory)
- [Bulk User Creation](#bulk-user-creation)
- [Service Account Patterns](#service-account-patterns)
- [Scripting with useradd](#scripting-with-useradd)

---

## Basic User Creation

```bash
# Minimal: creates user with no home, no password, default shell
useradd alice

# Standard: create with home directory (most common)
useradd -m alice

# Full: home + bash shell + full name
useradd -m -s /bin/bash -c "Alice Smith" alice

# After creation, always set a password:
passwd alice

# Or set password in one step (less secure — hash visible in ps):
useradd -m -p "$(openssl passwd -6 'password123')" alice

# Verify creation:
id alice                          # uid, gid, groups
grep "^alice:" /etc/passwd        # passwd entry
grep "^alice:" /etc/shadow        # shadow entry (root only)
ls -la /home/alice                # home directory
```

---

## System Accounts

System accounts are for services/daemons — low UID, no login, no home by default.

```bash
# Basic system account (no home, no login shell)
useradd -r nginx

# System account with explicit settings
useradd -r \
  -s /sbin/nologin \
  -d /var/lib/myapp \
  -c "MyApp Service Account" \
  myapp

# System account with home directory (for service data)
useradd -r -m \
  -d /var/lib/postgresql \
  -s /bin/bash \
  -c "PostgreSQL Server" \
  postgres

# System account with specific UID (useful for containers — consistent UID across nodes)
useradd -r -u 999 \
  -s /sbin/nologin \
  -d /var/lib/redis \
  redis

# Verify system account UID is in system range:
id nginx
# uid=998(nginx) gid=998(nginx) groups=998(nginx)

# Lock system accounts (extra security — can't log in even with password):
passwd -l nginx     # lock password
usermod -s /sbin/nologin nginx   # disable shell
```

---

## Custom UID & GID

```bash
# Specify UID manually
useradd -u 1500 -m alice

# Specify GID (primary group must exist)
groupadd -g 2000 developers
useradd -u 1500 -g developers -m alice

# Specify both UID and GID matching (common in containers/NFS)
groupadd -g 1001 alice
useradd -u 1001 -g 1001 -m alice

# Allow non-unique UID (two users share same UID — rare, special use)
useradd -u 1001 -o duplicate_alice

# Find next available UID in a range:
getent passwd | awk -F: '$3 >= 1000 && $3 < 2000 {print $3}' \
  | sort -n | awk 'BEGIN{n=1000} {if($1==n)n++} END{print n}'

# Force specific UID range (override login.defs for this command):
useradd -K UID_MIN=5000 -K UID_MAX=6000 -m alice
```

---

## Groups & Permissions

```bash
# Add to supplementary groups at creation
useradd -m -G sudo,docker,www-data alice

# Add to groups (comma-separated, no spaces)
useradd -m -G audio,video,plugdev,netdev alice

# Don't create a private group (use default group from login.defs)
useradd -N -m alice       # -N: no user-private group
# alice's primary group = GROUP from /etc/default/useradd (usually 'users' or 100)

# Create user without private group, assign to existing group
useradd -N -g staff -m alice

# After creation: add to more groups
usermod -aG docker alice     # -a: append (don't remove from other groups)
usermod -aG sudo alice

# View group membership:
groups alice
id alice
getent group docker | grep alice
```

---

## Home Directory Control

```bash
# Create home directory (copies /etc/skel)
useradd -m alice
# → creates /home/alice, owned alice:alice, mode 755 (or per UMASK in login.defs)

# Custom home directory path
useradd -m -d /data/users/alice alice
useradd -m -d /srv/alice alice

# Don't create home directory (for accounts that don't need one)
useradd -M alice             # or just: useradd alice (default varies by distro)

# Use /dev/null as home (for service accounts)
useradd -r -d /nonexistent -s /sbin/nologin myservice

# Change base directory (instead of /home):
useradd -b /data/users -m alice
# → creates /data/users/alice

# Set home directory permissions (via login.defs UMASK):
grep UMASK /etc/login.defs
# UMASK 022   → home dir mode = 755
# UMASK 077   → home dir mode = 700 (more private)

# Override umask for one user:
useradd -m alice
chmod 700 /home/alice    # manually restrict after creation
```

---

## Shell & GECOS

```bash
# Set login shell
useradd -m -s /bin/bash alice       # bash
useradd -m -s /bin/zsh alice        # zsh
useradd -m -s /bin/fish alice       # fish
useradd -m -s /sbin/nologin alice   # no interactive login
useradd -m -s /bin/false alice      # no login (returns false)
useradd -m -s /usr/bin/git-shell alice   # git-only shell

# Check available shells:
cat /etc/shells

# GECOS field: full name, room, work phone, home phone, other
useradd -c "Alice Smith" -m alice
useradd -c "Alice Smith,Room 101,555-1234,555-5678,Notes" -m alice

# Change shell after creation:
chsh -s /bin/zsh alice
usermod -s /bin/zsh alice

# Change GECOS after creation:
chfn -f "Alice Smith" -r "101" -w "555-1234" alice
usermod -c "Alice Smith (Updated)" alice

# View GECOS:
getent passwd alice
finger alice            # if finger is installed
```

---

## Account Expiry & Aging

```bash
# Set account expiry date (account disabled on this date)
useradd -m -e 2024-12-31 alice    # expires Dec 31 2024

# Set inactivity period (days after password expiry before account locked)
useradd -m -f 30 alice    # lock 30 days after password expires

# Set both:
useradd -m -e 2024-12-31 -f 14 contractor_alice

# Force password change at first login:
useradd -m alice
passwd -e alice           # expire password immediately → must change at login

# Set password aging at creation (via chage after useradd):
useradd -m alice
chage -M 90 -m 7 -W 14 -I 30 alice
# -M 90: max password age 90 days
# -m 7:  min 7 days between changes
# -W 14: warn 14 days before expiry
# -I 30: disable 30 days after expiry

# Temporary contractor account (expires + forced password change):
useradd -m \
  -e $(date -d "+90 days" +%Y-%m-%d) \
  -c "Contractor $(date +%Y)" \
  -s /bin/bash \
  contractor_alice
passwd -e contractor_alice   # force password set at first login

# View aging info:
chage -l alice
```

---

## Skeleton Directory

```bash
# Default: copies /etc/skel to home
useradd -m alice
ls -la /home/alice
# .bash_logout  .bashrc  .profile

# Use alternative skeleton:
useradd -m -k /etc/skel.developer alice    # developer template
useradd -m -k /etc/skel.restricted alice   # restricted template

# Empty home (no skeleton files):
useradd -m -k /dev/null alice

# Customize skeleton for all new users:
# Add company bashrc:
cat >> /etc/skel/.bashrc << 'EOF'
# Company standard aliases
alias ll='ls -la'
alias update='sudo apt update && sudo apt upgrade'
export PATH="$PATH:$HOME/bin"
EOF

# Add default config files:
mkdir /etc/skel/.config
mkdir /etc/skel/bin
cp company_vimrc /etc/skel/.vimrc

# Create department-specific skeletons:
cp -r /etc/skel /etc/skel.ops
echo "export KUBECONFIG=~/.kube/config" >> /etc/skel.ops/.bashrc

cp -r /etc/skel /etc/skel.dev
echo "export NVM_DIR=~/.nvm" >> /etc/skel.dev/.bashrc
```

---

## Bulk User Creation

```bash
# Method 1: newusers command (built for bulk creation)
# Format: username:password:UID:GID:GECOS:home:shell
cat > /tmp/newusers.txt << EOF
alice:Password123!:1001:1001:Alice Smith:/home/alice:/bin/bash
bob:Password456!:1002:1002:Bob Jones:/home/bob:/bin/bash
carol:Password789!:1003:1003:Carol White:/home/carol:/bin/bash
EOF

newusers /tmp/newusers.txt
shred -u /tmp/newusers.txt    # securely delete file with passwords


# Method 2: loop from CSV (username,fullname,shell)
while IFS=, read -r username fullname shell; do
  # Skip header
  [[ "$username" == "username" ]] && continue

  # Create user
  useradd -m \
    -s "$shell" \
    -c "$fullname" \
    "$username"

  # Generate random password
  temp_pass=$(tr -dc 'A-Za-z0-9!@#' < /dev/urandom | head -c 16)
  echo "$username:$temp_pass" | chpasswd

  # Force password change at first login
  passwd -e "$username"

  echo "Created: $username  Temp password: $temp_pass"
done < users.csv


# Method 3: from LDAP/CSV with group assignment
create_user() {
  local username="$1"
  local fullname="$2"
  local groups="$3"

  useradd -m -s /bin/bash -c "$fullname" "$username" || return 1
  usermod -aG "$groups" "$username"
  passwd -e "$username"    # force password at first login
  echo "✓ Created $username (groups: $groups)"
}

create_user "alice"  "Alice Smith"  "sudo,docker"
create_user "bob"    "Bob Jones"    "developers,docker"
create_user "carol"  "Carol White"  "analysts,readonly"


# Method 4: read from file with error handling
while IFS=: read -r user pass uid gid gecos home shell; do
  if id "$user" &>/dev/null; then
    echo "SKIP: $user already exists"
    continue
  fi

  useradd -u "$uid" -g "$gid" -m -d "$home" -s "$shell" -c "$gecos" "$user"
  echo "$user:$pass" | chpasswd

  if [ $? -eq 0 ]; then
    echo "OK: $user"
  else
    echo "FAIL: $user" >&2
  fi
done < /etc/newusers.conf
```

---

## Service Account Patterns

```bash
# Web server account (nginx/apache)
groupadd -r www-data
useradd -r \
  -g www-data \
  -d /var/www \
  -s /sbin/nologin \
  -c "Web Server" \
  www-data

# Database account (postgres/mysql)
useradd -r \
  -m \
  -d /var/lib/postgresql \
  -s /bin/bash \
  -c "PostgreSQL Database Server" \
  postgres

# Application account (runs java/python app)
groupadd -g 9001 myapp
useradd -r \
  -u 9001 \
  -g myapp \
  -m \
  -d /opt/myapp \
  -s /sbin/nologin \
  -c "MyApp Service" \
  myapp

# Create app directory structure:
mkdir -p /opt/myapp/{bin,conf,logs,data}
chown -R myapp:myapp /opt/myapp
chmod 750 /opt/myapp

# SSH-only git account
useradd -m \
  -s /usr/bin/git-shell \
  -c "Git Repository Account" \
  git

mkdir /home/git/.ssh
touch /home/git/.ssh/authorized_keys
chmod 700 /home/git/.ssh
chmod 600 /home/git/.ssh/authorized_keys
chown -R git:git /home/git/.ssh

# Chrooted SFTP account
useradd -m \
  -d /var/sftp/uploads \
  -s /sbin/nologin \
  -c "SFTP Upload Account" \
  sftpuser

# Locked account for cron jobs only (no interactive login)
useradd -r \
  -s /sbin/nologin \
  -d /var/lib/cronservice \
  cronservice
passwd -l cronservice    # lock password explicitly
```

---

## Scripting with useradd

```bash
# Check if user exists before creating
if id "alice" &>/dev/null; then
  echo "User alice already exists"
else
  useradd -m -s /bin/bash alice
  echo "User alice created"
fi

# Idempotent user creation function
ensure_user() {
  local username="$1"
  local shell="${2:-/bin/bash}"
  local groups="${3:-}"

  if id "$username" &>/dev/null; then
    echo "User $username already exists — skipping"
    return 0
  fi

  useradd -m -s "$shell" "$username" || { echo "Failed to create $username" >&2; return 1; }

  if [ -n "$groups" ]; then
    usermod -aG "$groups" "$username" || echo "Warning: could not add $username to $groups"
  fi

  passwd -e "$username"   # force password change at first login
  echo "Created user: $username"
}

ensure_user "alice" "/bin/bash" "sudo,docker"
ensure_user "myservice" "/sbin/nologin" ""


# Verify user was created correctly
verify_user() {
  local user="$1"
  local errors=0

  id "$user" &>/dev/null || { echo "FAIL: $user not in passwd"; ((errors++)); }

  sudo grep -q "^$user:" /etc/shadow || { echo "FAIL: $user not in shadow"; ((errors++)); }

  [ -d "/home/$user" ] || { echo "FAIL: /home/$user not created"; ((errors++)); }

  [ "$(stat -c %U /home/$user)" = "$user" ] || { echo "FAIL: wrong home owner"; ((errors++)); }

  return $errors
}


# Delete user if creation fails (cleanup):
create_user_safe() {
  local user="$1"
  useradd -m -s /bin/bash "$user" || return 1

  # Additional setup
  if ! some_other_setup "$user"; then
    echo "Setup failed, removing user"
    userdel -r "$user"   # cleanup on failure
    return 1
  fi
}


# Audit: list all users created in last 7 days
find /home -maxdepth 1 -type d -newer /tmp/week_ago -not -path "/home" \
  | xargs -I{} basename {}
```

---

> See also: [`README.md`](README.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
