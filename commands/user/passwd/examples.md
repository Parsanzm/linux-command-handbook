# passwd — Practical Examples

> Real-world usage for sysadmins, DevOps, and security engineers.

---

## Table of Contents

- [Basic Password Change](#basic-password-change)
- [Root: Managing Other Users](#root-managing-other-users)
- [Account Locking & Unlocking](#account-locking--unlocking)
- [Password Aging & Policy](#password-aging--policy)
- [Forcing Password Change](#forcing-password-change)
- [Password Status](#password-status)
- [Non-interactive: chpasswd](#non-interactive-chpasswd)
- [Generating Password Hashes](#generating-password-hashes)
- [Bulk Operations](#bulk-operations)
- [Scripting with passwd](#scripting-with-passwd)

---

## Basic Password Change

```bash
# Change your own password (prompts for current, then new)
passwd

# Session:
# Changing password for alice.
# Current password:           ← type current (hidden)
# New password:               ← type new (hidden)
# Retype new password:        ← confirm
# passwd: password updated successfully
```

---

## Root: Managing Other Users

```bash
# Change another user's password (root only — no current password needed)
sudo passwd alice
# New password:
# Retype new password:
# passwd: password updated successfully

# Change root's own password
sudo passwd root
# or (when already root):
passwd

# Delete a user's password (makes account passwordless)
sudo passwd -d alice
# Warning: passwordless accounts can log in without any password!

# Set password for newly created user
sudo useradd bob
sudo passwd bob
```

---

## Account Locking & Unlocking

```bash
# Lock an account (prepends ! to hash in /etc/shadow)
sudo passwd -l alice

# Verify it's locked:
sudo grep "^alice:" /etc/shadow
# alice:!$6$rounds=5000$...$hash...:...
#        ↑ ! means locked

# Check status
sudo passwd -S alice
# alice L 2024-06-15 0 99999 7 -1

# Unlock the account
sudo passwd -u alice

# Verify unlocked:
sudo grep "^alice:" /etc/shadow
# alice:$6$rounds=5000$...$hash...:...
#        ↑ no ! = unlocked

# Lock account after too many failures (done by pam_faillock, not passwd)
# View faillock status:
faillock --user alice
# Reset faillock:
faillock --user alice --reset
```

**Note:** `-l` only blocks password-based login. SSH key authentication still works unless you also disable the shell or use `usermod -e`.

---

## Password Aging & Policy

```bash
# Set maximum password age (days before forced change)
sudo passwd -x 90 alice      # must change every 90 days

# Set minimum password age (days before user can change again)
sudo passwd -n 7 alice       # can't change more than once per week

# Set warning period (days before expiry to warn)
sudo passwd -w 14 alice      # warn 14 days before expiry

# Set inactivity period (days after expiry account is disabled)
sudo passwd -i 30 alice      # disable 30 days after password expires

# Set all aging at once (better: use chage for this)
sudo passwd -x 90 -n 7 -w 14 -i 30 alice

# Never expire (set max days to -1 or 99999)
sudo passwd -x -1 alice
sudo passwd -x 99999 alice

# View aging info (use chage for more detail)
sudo chage -l alice
# Last password change       : Jun 15, 2024
# Password expires           : Sep 13, 2024
# Password inactive          : Oct 13, 2024
# Account expires            : never
# Minimum number of days     : 7
# Maximum number of days     : 90
# Number of days of warning  : 14
```

---

## Forcing Password Change

```bash
# Force user to change password at next login
sudo passwd -e alice
# passwd: password expiry information changed.

# Verify:
sudo passwd -S alice
# alice P 2024-06-15 0 90 7 30
#         ↑ P = has password; the last-change date set to epoch forces change

sudo chage -l alice
# Password expires : Jun 15, 2024   ← set to today = expired = must change

# What happens when alice logs in:
# WARNING: Your password has expired.
# You must change your password now and login again!
# Changing password for alice.
# Current password:
```

---

## Password Status

```bash
# Show status for one user
sudo passwd -S alice
# Format: username status last-change min max warn inactive
# alice P 2024-06-15 7 90 14 30
#         ↑ Status codes:
#           P = has usable password
#           L = locked
#           NP = no password

# Show status for ALL users
sudo passwd -Sa
# or:
sudo passwd -S -a

# Detailed view (use chage instead):
sudo chage -l alice

# Check if account is locked (grep for ! in shadow)
sudo grep "^alice:" /etc/shadow | awk -F: '{print $2}' | grep -q "^!" && echo "LOCKED"

# List all locked accounts
sudo awk -F: '($2 ~ /^!/) {print $1, "LOCKED"}' /etc/shadow

# List accounts with no password
sudo awk -F: '($2 == "") {print $1, "NO PASSWORD"}' /etc/shadow

# List accounts with expired passwords
sudo chage -l $(awk -F: '{print $1}' /etc/passwd) 2>/dev/null | grep -B1 "password expires"
```

---

## Non-interactive: chpasswd

`passwd` is interactive. For scripts and automation, use `chpasswd`:

```bash
# Change one user's password non-interactively
echo "alice:newpassword123" | sudo chpasswd

# Change multiple users
sudo chpasswd << EOF
alice:Password123!
bob:SecurePass456!
carol:MyPass789!
EOF

# From a file
sudo chpasswd < passwords.txt
# passwords.txt format: username:plaintext_password (one per line)

# With encrypted passwords (pre-hashed)
sudo chpasswd -e << EOF
alice:$6$rounds=5000$salt$hashhashhashhash...
EOF
# -e means passwords are already hashed
```

---

## Generating Password Hashes

```bash
# Generate SHA-512 hash (compatible with /etc/shadow)
openssl passwd -6 "mypassword"
# $6$randomsalt$hashhashhashhash...

# Generate with specific salt
openssl passwd -6 -salt "mysalt12" "mypassword"

# Generate SHA-256 hash
openssl passwd -5 "mypassword"

# Generate MD5 (legacy, avoid)
openssl passwd -1 "mypassword"

# Using Python (more control)
python3 -c "import crypt; print(crypt.crypt('mypassword', crypt.mksalt(crypt.METHOD_SHA512)))"

# Using mkpasswd (from whois package)
mkpasswd -m sha-512 "mypassword"
mkpasswd -m sha-512 -S "mysalt12" "mypassword"

# Verify a known hash matches a password
python3 -c "
import crypt
stored = '\$6\$salt\$hash...'
attempt = 'mypassword'
print('Match' if crypt.crypt(attempt, stored) == stored else 'No match')
"
```

---

## Bulk Operations

```bash
# Create users with passwords from a CSV (username,password)
while IFS=, read -r username password; do
  useradd -m "$username" 2>/dev/null
  echo "$username:$password" | chpasswd
  echo "Created: $username"
done < users.csv

# Force all users to change password at next login
for user in $(awk -F: '$3 >= 1000 && $3 < 65534 {print $1}' /etc/passwd); do
  sudo passwd -e "$user"
  echo "Expired: $user"
done

# Lock all users except root and system accounts
for user in $(awk -F: '$3 >= 1000 && $3 < 65534 {print $1}' /etc/passwd); do
  sudo passwd -l "$user"
  echo "Locked: $user"
done

# Set 90-day expiry for all regular users
for user in $(awk -F: '$3 >= 1000 && $3 < 65534 {print $1}' /etc/passwd); do
  sudo passwd -x 90 -w 14 "$user"
done

# Unlock all users
for user in $(awk -F: '$3 >= 1000 && $3 < 65534 {print $1}' /etc/passwd); do
  sudo passwd -u "$user" 2>/dev/null
done

# Report status of all regular users
echo "USERNAME STATUS LAST_CHANGE MAX_DAYS"
for user in $(awk -F: '$3 >= 1000 && $3 < 65534 {print $1}' /etc/passwd); do
  sudo passwd -S "$user"
done
```

---

## Scripting with passwd

```bash
# Check if a user's account is locked
is_locked() {
  local user="$1"
  sudo grep "^$user:" /etc/shadow | awk -F: '{print $2}' | grep -q "^!"
}

if is_locked alice; then
  echo "alice is locked"
fi

# Check if account has no password
has_no_password() {
  local user="$1"
  local hash
  hash=$(sudo grep "^$user:" /etc/shadow | awk -F: '{print $2}')
  [ -z "$hash" ]
}

# Generate a random password
generate_password() {
  local length="${1:-16}"
  tr -dc 'A-Za-z0-9!@#$%^&*' < /dev/urandom | head -c "$length"
  echo
}

new_pass=$(generate_password 20)
echo "alice:$new_pass" | sudo chpasswd
echo "New password for alice: $new_pass"

# Reset password and force change at next login
reset_user_password() {
  local user="$1"
  local temp_pass
  temp_pass=$(generate_password 12)
  echo "$user:$temp_pass" | sudo chpasswd
  sudo passwd -e "$user"
  echo "Temp password for $user: $temp_pass (must change at next login)"
}

# Validate password complexity before setting (bash)
validate_password() {
  local pass="$1"
  local errors=0

  [ ${#pass} -lt 12 ] && echo "Too short (min 12)" && ((errors++))
  echo "$pass" | grep -q "[A-Z]" || { echo "No uppercase"; ((errors++)); }
  echo "$pass" | grep -q "[a-z]" || { echo "No lowercase"; ((errors++)); }
  echo "$pass" | grep -q "[0-9]" || { echo "No digit"; ((errors++)); }
  echo "$pass" | grep -q "[^A-Za-z0-9]" || { echo "No special char"; ((errors++)); }

  return $errors
}
```

---

> See also: [`README.md`](README.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
