# useradd — Interview Questions

> Covers beginner to senior level. Each question includes the answer.
> Questions marked 🔥 are commonly asked in real interviews.

---

## Table of Contents

- [Conceptual](#conceptual)
- [Basic Usage](#basic-usage)
- [Files & Databases](#files--databases)
- [UID, GID & Groups](#uid-gid--groups)
- [System Accounts](#system-accounts)
- [Security & Hardening](#security--hardening)
- [Scripting & Automation](#scripting--automation)
- [Scenario-Based](#scenario-based)
- [Advanced & Internals](#advanced--internals)

---

## Conceptual

**Q1 🔥 What files does `useradd` modify?**
> `useradd` modifies up to four files:
> 1. `/etc/passwd` — adds user entry (username, UID, GID, GECOS, home, shell)
> 2. `/etc/shadow` — adds password entry (with `!!` = no password set)
> 3. `/etc/group` — creates primary group (or adds to existing)
> 4. `/etc/gshadow` — adds group shadow entry
>
> Additionally, if `-m` is used: creates home directory and copies `/etc/skel` contents.

---

**Q2. What is the difference between `useradd` and `adduser`?**
> - `useradd` — low-level C binary, same on all Linux distros, fully non-interactive, does exactly what flags specify, no prompts, no default password, doesn't always create home dir.
> - `adduser` — high-level Perl script (Debian/Ubuntu only), interactive, always creates home, prompts for password and GECOS, more user-friendly but not universal.
>
> In scripts: always use `useradd`. For manual admin on Debian: `adduser` is more convenient.

---

**Q3 🔥 Why is there no password set after `useradd`?**
> `useradd` creates the account with `!!` in the shadow password field — meaning the account is locked with no password. This is intentional: the account is created but cannot be used until a password is explicitly set with `passwd username`. This prevents accidental creation of passwordless accounts.

---

**Q4. What does the `-r` flag do?**
> Creates a **system account** — an account intended for services/daemons rather than interactive users:
> - UID allocated from the system range (`SYS_UID_MIN`-`SYS_UID_MAX`, default 201-999)
> - No home directory created by default
> - No password aging applied
> - No entry added to `/var/mail`
>
> `-r` does **not** automatically set a no-login shell or lock the account — you must add `-s /sbin/nologin` explicitly.

---

**Q5. What is `/etc/skel` and what is its role in `useradd`?**
> `/etc/skel` (skeleton directory) contains template files copied into every new user's home directory when `useradd -m` is used. Typically contains `.bashrc`, `.bash_profile`, `.bash_logout`, `.profile`. Administrators customize `/etc/skel` to give all new users a standard environment. Use `-k /alternative/skel` to use a different skeleton.

---

## Basic Usage

**Q6 🔥 What is the minimum command to create a fully usable user account?**
> ```bash
> useradd -m -s /bin/bash alice
> passwd alice
> ```
> `-m` creates the home directory. Without a shell specified, the default may be `/bin/sh` (Debian). Without `passwd`, the account is locked and unusable.

---

**Q7. How do you create a user and add them to multiple groups simultaneously?**
> ```bash
> useradd -m -G sudo,docker,www-data alice
> # -G: supplementary groups (comma-separated, no spaces)
> # Primary group is created automatically (alice:alice) by default
> ```

---

**Q8 🔥 What is the difference between `-m` and `-M`?**
> - `-m` — **create** home directory (and copy `/etc/skel`)
> - `-M` — explicitly **do NOT create** home directory
>
> On Debian/Ubuntu, the default is no home directory (as if `-M` were set). On RHEL/Fedora, the default is to create home (`CREATE_HOME yes` in `/etc/login.defs`). Always specify `-m` explicitly for portability.

---

**Q9. How do you create a user with a specific home directory path?**
> ```bash
> useradd -m -d /data/users/alice alice
> # -d: sets the home directory path
> # -m: actually creates it (without -m, the path is recorded but directory not created)
> ```

---

**Q10. How do you set an account expiry date at creation?**
> ```bash
> useradd -m -e 2024-12-31 alice       # expires Dec 31, 2024
> useradd -m -e $(date -d "+90 days" +%Y-%m-%d) contractor  # expires in 90 days
> ```
> After the expiry date, the account is disabled regardless of password validity.

---

## Files & Databases

**Q11 🔥 Explain the fields in `/etc/passwd`.**
> ```
> alice:x:1001:1001:Alice Smith:/home/alice:/bin/bash
> ```
> 1. Username
> 2. Password: `x` means shadow is used (actual hash is in `/etc/shadow`)
> 3. UID
> 4. Primary GID
> 5. GECOS: comment field (full name, phone, etc.)
> 6. Home directory
> 7. Login shell

---

**Q12 🔥 What does `!!` mean in `/etc/shadow` after `useradd`?**
> `!!` means the account has **no password set** and is **locked**. The double `!` is a convention indicating the account was never given a valid password. A single `!` (from `passwd -l`) means a password exists but was locked. An account with `!!` can't log in with a password until `passwd username` is run.

---

**Q13. What is `/etc/login.defs` and which `useradd` behavior does it control?**
> `/etc/login.defs` provides system-wide policy for user account creation:
> - `UID_MIN` / `UID_MAX` — range for regular user UIDs
> - `SYS_UID_MIN` / `SYS_UID_MAX` — range for system account UIDs
> - `PASS_MAX_DAYS` — default max password age for new accounts
> - `PASS_MIN_DAYS` — default min days between password changes
> - `PASS_WARN_AGE` — default warning days before expiry
> - `ENCRYPT_METHOD` — hashing algorithm (SHA512, YESCRYPT)
> - `CREATE_HOME` — whether to create home by default
> - `UMASK` — permissions mask for home directory

---

**Q14. What is `/etc/default/useradd` and how do you view its settings?**
> Contains defaults used by `useradd` when no flags are specified:
> ```bash
> useradd -D       # view current defaults
> cat /etc/default/useradd
> ```
> Key settings: `SHELL`, `HOME` (base dir), `INACTIVE`, `EXPIRE`, `SKEL`, `CREATE_MAIL_SPOOL`.
>
> Modify defaults:
> ```bash
> useradd -D -s /bin/bash    # change default shell
> ```

---

**Q15. What does `pwck` do and when should you run it?**
> `pwck` verifies the integrity and consistency of `/etc/passwd` and `/etc/shadow`:
> - Every entry in `passwd` has a corresponding entry in `shadow`
> - All UIDs are unique
> - Home directories exist
> - Login shells are valid
> - No corrupted entries
>
> Run after: manual edits to passwd/shadow, failed `useradd`, suspected file corruption, or as part of security audits.

---

## UID, GID & Groups

**Q16 🔥 What is the difference between primary group and supplementary groups?**
> - **Primary group** (`-g`): assigned at account creation, stored as GID in `/etc/passwd`. Files created by the user belong to this group by default.
> - **Supplementary groups** (`-G`): additional group memberships stored in `/etc/group`. Give access to resources owned by those groups.
>
> ```bash
> useradd -g staff -G sudo,docker alice
> # Primary: staff (alice's files owned by staff group)
> # Supplementary: sudo, docker (alice can use sudo and run docker)
> ```

---

**Q17 🔥 Why is `usermod -G docker alice` dangerous? What's the safe alternative?**
> `usermod -G docker alice` **replaces** all supplementary groups with just `docker`. Alice loses sudo, audio, video, and any other group memberships.
>
> Safe: always use `-aG` to append:
> ```bash
> usermod -aG docker alice    # adds docker, keeps all other groups
> ```

---

**Q18. What happens to files when a user is deleted and their UID is reused?**
> Files on disk store numeric UIDs, not usernames. If user alice (UID 1001) is deleted and new user bob gets UID 1001:
> - All of alice's old files (if not deleted) now appear owned by bob
> - Bob gains unintended access to alice's data
>
> Prevention:
> ```bash
> userdel -r alice    # remove home directory with the user
> find / -uid 1001 2>/dev/null    # find any remaining orphaned files
> ```

---

**Q19. What is a private user group and why is it the default?**
> When `useradd` creates a user, it also creates a group with the same name and GID as the user's UID (e.g., user `alice` with GID 1001 and group `alice`). This is the **User Private Group (UPG)** scheme.
>
> Benefits:
> - Each user's files are private by default (no accidental sharing)
> - Collaboration is opt-in via `chmod g+rw` + `newgrp`
> - Avoids the old model where all users shared the `users` group
>
> Disable with `-N` (no user group, uses default GID from `/etc/default/useradd`).

---

## System Accounts

**Q20 🔥 What is a system account and how do you create one properly?**
> A system account is for services/daemons — not for human login. Proper creation:
> ```bash
> useradd -r \
>   -s /sbin/nologin \
>   -d /var/lib/myservice \
>   -c "MyService Daemon" \
>   myservice
> ```
> - `-r`: allocate UID from system range (< 1000)
> - `-s /sbin/nologin`: prevent interactive login
> - `-d`: set data directory (not `/home`)
> - No `-m`: don't create directory (create manually with correct permissions)

---

**Q21. What is `/sbin/nologin` and how does it differ from `/bin/false`?**
> Both prevent interactive login, but:
> - `/sbin/nologin` — prints "This account is currently not available." and exits 1. More informative for debugging.
> - `/bin/false` — exits 1 immediately, no message.
>
> For SFTP-only accounts, some configurations require `/sbin/nologin` (not `/bin/false`). For pure service accounts, either works; `/sbin/nologin` is preferred.

---

**Q22. Does `-r` prevent a system account from logging in?**
> No — `-r` only controls which UID range is used (system vs regular). The account can still log in if it has a valid shell and password.
>
> To truly prevent login: use `-s /sbin/nologin` and don't set a password (or use `passwd -l` to lock).

---

## Security & Hardening

**Q23 🔥 What security risk exists when using `useradd -p`?**
> The `-p` flag takes a pre-hashed password. During execution, the hash appears in the process list (`ps aux`), which is readable by all users on the system. An attacker can capture the hash and attempt offline cracking.
>
> Safer approaches:
> ```bash
> useradd -m alice && echo "alice:password" | chpasswd    # pipe (still briefly visible)
> useradd -m alice && passwd alice                         # interactive (most secure)
> ```

---

**Q24. What `/etc/login.defs` settings should be reviewed for security hardening?**
> Key settings for CIS Benchmark / security compliance:
> ```ini
> PASS_MAX_DAYS   90      # max 90 days (not 99999)
> PASS_MIN_DAYS   7       # can't change more than once per week
> PASS_WARN_AGE   14      # 2-week warning
> PASS_MIN_LEN    12      # minimum length
> ENCRYPT_METHOD  SHA512  # or YESCRYPT (not MD5!)
> UMASK           077     # private home directories (700)
> ```

---

**Q25. How do you ensure all new users must change their password at first login?**
> Two approaches:
> ```bash
> # Per-user (after creation):
> useradd -m alice
> passwd -e alice          # expire immediately → forced change at login
>
> # System-wide (for all new users):
> # Set PASS_MAX_DAYS 1 in /etc/login.defs temporarily,
> # then reset after creation. Or use chage defaults.
>
> # Better: use chage for fine control:
> useradd -m alice
> chage -d 0 alice         # set last-change to epoch → must change immediately
> ```

---

## Scripting & Automation

**Q26 🔥 How do you create a user idempotently in a script (don't fail if user exists)?**
> ```bash
> if ! id "alice" &>/dev/null; then
>   useradd -m -s /bin/bash -c "Alice Smith" alice
>   echo "alice:$(openssl rand -base64 12)" | chpasswd
>   passwd -e alice
> else
>   echo "User alice already exists — skipping"
> fi
> ```

---

**Q27. How do you create 50 users from a CSV file efficiently?**
> ```bash
> # CSV format: username,fullname,groups
> while IFS=, read -r user fullname groups; do
>   [[ "$user" == "username" ]] && continue   # skip header
>   id "$user" &>/dev/null && { echo "SKIP: $user"; continue; }
>
>   useradd -m -s /bin/bash -c "$fullname" "$user"
>   usermod -aG "$groups" "$user"
>
>   temp_pass=$(tr -dc 'A-Za-z0-9!@#' < /dev/urandom | head -c 16)
>   echo "$user:$temp_pass" | chpasswd
>   passwd -e "$user"
>   echo "Created $user — temp: $temp_pass"
> done < users.csv
> ```

---

**Q28. What is `newusers` and when is it better than a `useradd` loop?**
> `newusers` reads a file of `username:password:UID:GID:GECOS:home:shell` lines and creates all users atomically. Better for:
> - Large batch imports (faster, less overhead)
> - When UIDs/GIDs must be exact
> - When you have pre-hashed passwords
>
> ```bash
> newusers /tmp/users.txt
> ```
>
> Downsides: file contains plaintext passwords (secure it!), less flexible than a loop (no per-user logic).

---

## Scenario-Based

**Q29 🔥 A new employee joins. What is the complete set of commands to properly set up their account?**
> ```bash
> # 1. Create account
> useradd -m -s /bin/bash -c "Alice Smith" alice
>
> # 2. Set temporary password
> echo "alice:TempPass123!" | chpasswd
>
> # 3. Force password change at first login
> passwd -e alice
>
> # 4. Add to appropriate groups
> usermod -aG sudo,docker alice
>
> # 5. Set password aging policy
> chage -M 90 -W 14 -m 7 alice
>
> # 6. Verify everything
> id alice
> chage -l alice
> ls -la /home/alice
> ```

---

**Q30 🔥 After running `useradd alice`, the user complains they can't log in. What are the possible causes?**
> Three most likely causes:
> 1. **No password set**: `!!` in shadow = locked. Fix: `passwd alice`
> 2. **No home directory**: `-m` was not used. Fix: `mkdir /home/alice && cp -r /etc/skel/. /home/alice/ && chown -R alice:alice /home/alice`
> 3. **Invalid/missing shell**: shell specified doesn't exist or isn't in `/etc/shells`. Fix: `usermod -s /bin/bash alice`
>
> Debugging:
> ```bash
> sudo grep "^alice:" /etc/shadow   # check for !!
> ls /home/alice                    # check home exists
> getent passwd alice | cut -d: -f7 # check shell
> ```

---

**Q31. You need to create a service account for an application that should:**
> - Not log in interactively
> - Own files in `/opt/myapp`
> - Have a consistent UID of 9000 across all servers
> - Have no password
>
> ```bash
> # Create system account with specific UID
> useradd -r \
>   -u 9000 \
>   -s /sbin/nologin \
>   -d /opt/myapp \
>   -c "MyApp Service Account" \
>   myapp
>
> # Create and set up data directory
> mkdir -p /opt/myapp/{bin,conf,logs,data}
> chown -R myapp:myapp /opt/myapp
> chmod 750 /opt/myapp
>
> # Verify UID is consistent
> id myapp
> # uid=9000(myapp) gid=9000(myapp) groups=9000(myapp)
>
> # Lock (no password, extra safety):
> passwd -l myapp
> ```

---

**Q32 🔥 You discover that `/etc/passwd` and `/etc/shadow` are inconsistent after a failed `useradd`. How do you fix it?**
> ```bash
> # 1. Check for inconsistencies
> pwck
> # Output: user 'alice': no corresponding entry in /etc/shadow
>
> # 2. Option A: fix automatically
> pwck -s    # sort and fix (interactive)
>
> # 3. Option B: manually add missing shadow entry
> # Use vipw -s to safely edit /etc/shadow
> vipw -s
> # Add line: alice:!!:$(date -d "today" +%s)/86400:0:99999:7:::
>
> # 4. Option C: remove the partial entry and recreate
> vipw        # remove alice from /etc/passwd
> vigr        # remove alice from /etc/group
> useradd -m alice   # recreate clean
>
> # 5. Convert all passwd entries to shadow (if shadow is missing multiple):
> pwconv
> ```

---

## Advanced & Internals

**Q33. How does `useradd` prevent corruption of `/etc/passwd` when multiple instances run simultaneously?**
> `useradd` calls `lckpwdf()` which creates a lock file `/etc/passwd.lock` (or uses fcntl locking). Only one process can hold the lock at a time. If the lock can't be acquired (another `useradd`/`usermod`/`userdel` is running), the command fails with "cannot lock /etc/passwd; try again later." This prevents two processes from writing to `/etc/passwd` simultaneously and corrupting it.

---

**Q34. What is the GECOS field and what format does it use?**
> GECOS (General Electric Comprehensive Operating Supervisor — a historical reference) is the comment/info field in `/etc/passwd`. Traditional format:
> ```
> Full Name,Room Number,Work Phone,Home Phone,Other
> alice:...:Alice Smith,101,555-1234,555-5678,Notes
> ```
> Most modern systems only use the full name portion. The `finger` command displays GECOS information. Set with `-c` on `useradd`, modified with `chfn` or `usermod -c`.

---

**Q35 🔥 What is the difference between account expiry (`-e`) and password expiry (`chage -M`)?**
> - **Account expiry** (`-e DATE` / `chage -E DATE`): the entire account becomes inaccessible after this date, regardless of whether the password is valid. Stored in field 8 of `/etc/shadow`.
> - **Password expiry** (`-x DAYS` / `chage -M DAYS`): the password must be changed after this many days. The account is still active — the user is just forced to set a new password. Stored in field 5 of `/etc/shadow`.
>
> Both are stored in `/etc/shadow`; both can disable access, but through different mechanisms.

---

> See also: [`README.md`](README.md) · [`examples.md`](examples.md) · [`edge-cases.md`](edge-cases.md)
