# passwd — Interview Questions

> Covers beginner to senior level. Each question includes the answer.
> Questions marked 🔥 are commonly asked in real interviews.

---

## Table of Contents

- [Conceptual](#conceptual)
- [SUID & Security](#suid--security)
- [Password Storage](#password-storage)
- [Account Management](#account-management)
- [Scripting & Automation](#scripting--automation)
- [Scenario-Based](#scenario-based)
- [Advanced](#advanced)

---

## Conceptual

**Q1 🔥 What does `passwd` do beyond just changing a password?**
> `passwd` manages the full lifecycle of a user's authentication:
> - Changes passwords (interactively or as root)
> - Locks and unlocks accounts (`-l`, `-u`)
> - Sets password aging policy (`-x`, `-n`, `-w`, `-i`)
> - Forces password expiry at next login (`-e`)
> - Deletes passwords (`-d`)
> - Reports account status (`-S`)
>
> It's a complete account authentication management tool, not just a password changer.

---

**Q2. Where does `passwd` store the new password?**
> In `/etc/shadow` — the shadow password file. The password is **hashed** (not stored in plaintext) using an algorithm like SHA-512 or yescrypt. `/etc/shadow` is readable only by root (permissions: `640`, owner: `root`, group: `shadow`).

---

**Q3 🔥 What is the difference between `/etc/passwd` and `/etc/shadow`?**
> `/etc/passwd` — world-readable user database: username, UID, GID, home directory, shell. The password field contains `x` (meaning: look in shadow).
>
> `/etc/shadow` — root-only password file: hashed passwords, aging info, lockout status. Separated from `/etc/passwd` specifically so hashes aren't exposed to all users.
>
> This separation is called the "shadow password suite" — introduced because the original system stored hashes in `/etc/passwd`, which was world-readable and vulnerable to offline cracking.

---

**Q4. How does `passwd` know whether it's being run by root or a regular user?**
> It calls `getuid()` (real UID, not effective UID). Even though `passwd` runs with effective UID 0 (root) due to SUID, `getuid()` returns the UID of who actually invoked it. If `getuid() != 0`, `passwd` enforces: must provide current password, can only change own password, new password subject to policy.

---

**Q5. What happens to the old password when you change it?**
> It's replaced in `/etc/shadow`. The old hash is overwritten — `passwd` itself doesn't archive it. However, if `pam_pwhistory` is configured, PAM saves old hashes to `/etc/security/opasswd` to prevent reuse.

---

## SUID & Security

**Q6 🔥 Why does `passwd` have the SUID bit? What would break without it?**
> `/etc/shadow` is only writable by root. For a regular user to change their own password, `passwd` must be able to write to `/etc/shadow`. The SUID bit makes `passwd` run with the file owner's UID (root), giving it write access. Without SUID, regular users would get "Permission denied" when trying to change their own password.

---

**Q7 🔥 Explain the difference between `getuid()` and `geteuid()` in the context of `passwd`.**
> - `getuid()` — **real** UID: the user who actually invoked the process.
> - `geteuid()` — **effective** UID: the UID used for permission checks (affected by SUID).
>
> For `passwd` run by alice (UID 1001): `getuid()` = 1001, `geteuid()` = 0 (root, due to SUID). `passwd` uses `getuid()` to determine what the caller is allowed to do, while `geteuid()` (root) gives it the privilege to actually write `/etc/shadow`.

---

**Q8. What does the `s` in `-rwsr-xr-x` mean for `/usr/bin/passwd`?**
> It's the **SUID bit** in the owner's execute position. When this file is executed, the process runs with the **file owner's UID** (root) instead of the caller's UID. The lowercase `s` means SUID is set AND the owner has execute permission. An uppercase `S` would mean SUID is set but execute is NOT (unusual and usually a mistake).

---

**Q9. How does locking an account (`passwd -l`) work at the file level?**
> It prepends a `!` to the password hash in `/etc/shadow`:
> ```
> Before: alice:$6$salt$hash:...
> After:  alice:!$6$salt$hash:...
> ```
> When the system tries to verify a password, it computes the hash and compares. The `!` prefix guarantees no computed hash will ever match (since `!` is never a valid hash prefix), making login impossible. The original hash is preserved — `passwd -u` simply removes the `!` to restore it.

---

**Q10. What is the risk of SUID on `passwd` if the binary is compromised?**
> If an attacker replaces `/usr/bin/passwd` with a malicious binary, it runs as root when invoked by any user. The attacker could: read `/etc/shadow`, modify any account, create backdoors, or execute arbitrary root commands. This is why binary integrity checking (e.g., `dpkg -V`, `rpm -V`, or IMA/IMA-appraisal) is critical for SUID binaries.

---

## Password Storage

**Q11 🔥 What does a `/etc/shadow` entry look like and what does each field mean?**
> ```
> alice:$6$salt$hash:19523:0:90:14:30:19900:
> │     │              │    │  │   │  │   │
> │     │              │    │  │   │  │   └─ account expiry (days since epoch; empty=never)
> │     │              │    │  │   │  └───── inactive days after password expires
> │     │              │    │  │   └──────── warn days before expiry
> │     │              │    │  └──────────── max days between changes
> │     │              │    └─────────────── min days between changes
> │     │              └──────────────────── last password change (days since epoch)
> │     └─────────────────────────────────── hashed password
> └───────────────────────────────────────── username
> ```

---

**Q12 🔥 What do the `$6$`, `$5$`, `$1$`, and `$y$` prefixes mean in a password hash?**
> They identify the hashing algorithm:
> - `$1$` — MD5 (obsolete, insecure — rainbow tables exist)
> - `$5$` — SHA-256 (acceptable)
> - `$6$` — SHA-512 (current default on most Linux distros)
> - `$y$` — yescrypt (modern, Ubuntu 22.04+, more resistant to GPU cracking)
>
> yescrypt and bcrypt are memory-hard — they require significant RAM in addition to CPU, making GPU-based cracking much harder.

---

**Q13. What does `!!` mean in the password field of `/etc/shadow`?**
> The account was created but **never had a password set**. It's different from `!` (which is a locked account with no hash). `!!` means the account can't log in with a password because there's no hash to compare against. Usually seen right after `useradd` before `passwd` is run.

---

**Q14. How would you verify that a plaintext password matches a stored hash?**
> ```python
> import crypt
> stored_hash = "$6$salt$hash..."
> attempt = "mypassword"
> result = crypt.crypt(attempt, stored_hash)
> print("Match" if result == stored_hash else "No match")
> ```
> Or with OpenSSL:
> ```bash
> openssl passwd -6 -salt "extractedsalt" "mypassword"
> # Compare output with stored hash
> ```

---

## Account Management

**Q15 🔥 What is the difference between `passwd -l` and `usermod -e`?**
> - `passwd -l` — **locks the password**: prepends `!` to hash. Blocks password-based login only. SSH keys, su by root, and existing sessions still work.
> - `usermod -e DATE` — **sets account expiry date**: the account becomes completely inaccessible after that date regardless of authentication method. Setting to a past date immediately disables the account.
>
> For full account disable: use both together, plus `usermod -s /sbin/nologin`.

---

**Q16. How do you force a user to change their password at next login?**
> ```bash
> sudo passwd -e username
> ```
> This sets the "last password change" field in `/etc/shadow` to day 0 (epoch), which is treated as "immediately expired." When the user logs in, they're prompted to change password before getting a shell.

---

**Q17. What does `passwd -S` output mean?**
> ```
> alice P 2024-06-15 7 90 14 30
> │     │ │          │  │  │  │
> │     │ │          │  │  │  └─ inactive days after expiry
> │     │ │          │  │  └──── warn days before expiry
> │     │ │          │  └─────── max days between changes
> │     │ │          └────────── min days between changes
> │     │ └───────────────────── last password change date
> │     └─────────────────────── P=has password, L=locked, NP=no password
> └───────────────────────────── username
> ```

---

**Q18. What is the difference between `passwd -d` and `passwd -l`?**
> - `passwd -d` — **deletes** the password (empty hash field). The account may be able to log in **without a password** depending on PAM config (`nullok` option). Intended for accounts using only SSH keys, where you want to explicitly disallow password login.
> - `passwd -l` — **locks** the account by prepending `!` to the existing hash. The hash is preserved. No login is possible until unlocked with `passwd -u`.

---

## Scripting & Automation

**Q19 🔥 How do you change a password non-interactively in a script?**
> Use `chpasswd`, not `passwd`:
> ```bash
> echo "alice:newpassword123" | sudo chpasswd
> ```
> `passwd --stdin` works on RHEL/Fedora but not on Debian/Ubuntu. `chpasswd` is the portable, scripting-friendly tool.

---

**Q20. What's the security risk of running `echo "password" | passwd --stdin username` in a script?**
> Three risks:
> 1. **Process listing**: the password appears in `ps aux` output while the pipe is active
> 2. **Shell history**: the command may be saved in `.bash_history`
> 3. **Log files**: some auditing systems log command lines
>
> Safer: `echo "user:pass" | chpasswd` (same risk for the pass, but `chpasswd` is designed for this), or read from a protected file with `chpasswd < /root/passwords.txt && shred -u /root/passwords.txt`.

---

**Q21. How do you generate a secure random password in a bash script?**
> ```bash
> # Using /dev/urandom (most portable):
> tr -dc 'A-Za-z0-9!@#$%^&*' < /dev/urandom | head -c 20
>
> # Using openssl:
> openssl rand -base64 15
>
> # Using python:
> python3 -c "import secrets, string; print(secrets.token_urlsafe(15))"
> ```

---

## Scenario-Based

**Q22 🔥 A user is locked out of their account. How do you investigate and fix it?**
> ```bash
> # 1. Check password status
> sudo passwd -S alice
>
> # 2. Check if locked (! in shadow)
> sudo grep "^alice:" /etc/shadow | cut -d: -f2 | head -c1
> # → ! means locked
>
> # 3. Check for faillock (too many failed attempts)
> sudo faillock --user alice
>
> # Fix: unlock password
> sudo passwd -u alice
>
> # Fix: reset faillock
> sudo faillock --user alice --reset
>
> # Fix: if account expired
> sudo chage -E -1 alice   # -1 = never expires
> ```

---

**Q23 🔥 You need to set up 50 new user accounts with temporary passwords that must be changed at first login. How do you do this efficiently?**
> ```bash
> # Generate users.csv: username,fullname
> while IFS=, read -r user fullname; do
>   # Create user
>   useradd -m -c "$fullname" "$user"
>   # Generate temp password
>   temp_pass=$(tr -dc 'A-Za-z0-9' < /dev/urandom | head -c 12)
>   # Set password
>   echo "$user:$temp_pass" | chpasswd
>   # Force change at next login
>   passwd -e "$user"
>   echo "$user: $temp_pass"
> done < users.csv
> ```

---

**Q24. After a security breach, you need to force ALL users to reset their passwords immediately. What's the fastest approach?**
> ```bash
> # Expire all regular user accounts (UID 1000-65533)
> for user in $(awk -F: '$3 >= 1000 && $3 < 65534 {print $1}' /etc/passwd); do
>   passwd -e "$user"
>   echo "Expired: $user"
> done
> # Users will be forced to set new password at next login
> ```

---

**Q25. How do you check if any system accounts have a password set (security audit)?**
> ```bash
> # System accounts (UID < 1000) that have a real password hash
> sudo awk -F: '
>   NR==FNR {uid[$1]=$3; next}
>   ($2 ~ /^\$/) && (uid[$1]+0 < 1000) {print $1, "HAS PASSWORD HASH"}
> ' /etc/passwd /etc/shadow
> ```

---

**Q26 🔥 What's the difference between `passwd alice` and `chpasswd` when changing a user's password as root?**
> - `passwd alice` — interactive: prompts for new password twice, validates against PAM policy, logs to syslog.
> - `chpasswd` — non-interactive: reads `username:password` from stdin, still validates via PAM by default. Use `-e` if passing pre-hashed passwords. Better for scripts and bulk operations.
>
> Both write to `/etc/shadow`. `chpasswd` can be used in pipelines; `passwd` cannot without `--stdin` (RHEL only) or `expect`.

---

## Advanced

**Q27 🔥 What is PAM and how does `passwd` use it?**
> PAM (Pluggable Authentication Modules) is a framework that separates authentication logic from applications. `passwd` calls PAM for:
> 1. **Verifying old password** (`pam_unix`)
> 2. **Enforcing policy on new password** (`pam_pwquality`: length, complexity, dictionary)
> 3. **Checking history** (`pam_pwhistory`: preventing reuse)
> 4. **Writing the new hash** (`pam_unix` writes to `/etc/shadow`)
>
> Config: `/etc/pam.d/passwd` → typically includes `/etc/pam.d/common-password`.

---

**Q28. How does shadow password history (`pam_pwhistory`) work?**
> When a user changes their password, `pam_pwhistory` saves the old hash to `/etc/security/opasswd`. When a new password is set, it computes the hash and compares against stored history. If it matches any of the last N passwords, the change is rejected. Root bypasses this (or can use `chpasswd`/`usermod -p` to set hash directly).

---

**Q29. How do you increase the bcrypt/SHA-512 rounds for stronger hashing?**
> ```bash
> # In /etc/login.defs:
> SHA_CRYPT_MIN_ROUNDS 100000
> SHA_CRYPT_MAX_ROUNDS 100000
>
> # Or in /etc/pam.d/common-password:
> password ... pam_unix.so sha512 rounds=100000
>
> # Existing hashes are unaffected — users must change password to get new rounds
> # Force re-hash:
> sudo passwd -e username
> ```
> Higher rounds = more compute per hash attempt = slower offline cracking.

---

**Q30 🔥 A junior sysadmin accidentally ran `chmod u-s /usr/bin/passwd`. What broke and how do you fix it?**
> Without SUID, `passwd` runs with the caller's UID. Regular users (non-root) can no longer change their own passwords because `/etc/shadow` (owned by root, mode 640) is not writable by them. They get: `passwd: Authentication token manipulation error`.
>
> Root can still change passwords (root doesn't need SUID to write shadow).
>
> Fix:
> ```bash
> sudo chmod u+s /usr/bin/passwd
> # or:
> sudo chmod 4755 /usr/bin/passwd
> # Verify:
> ls -l /usr/bin/passwd
> # -rwsr-xr-x ...
> ```

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md)
