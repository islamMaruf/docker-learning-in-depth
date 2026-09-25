# Chapter 13: Users, Groups and Permissions

> **In one sentence:** Linux decides who may read, change or run every file using three ideas: **users** (who you are), **groups** (teams you belong to) and **permissions** (`r`, `w`, `x` for the owner, the group, and everyone else). Docker containers use the exact same system, and getting it wrong is behind many "permission denied" errors and security problems.

**Level:** 🟢 Beginner → 🟡 Intermediate → 🔴 Expert (Docker section) · **Reading time:** ~50 minutes

**Prerequisites:** [Chapter 12](12_linux_basic_commands.md). Practice safely in a container: `docker run -it --rm ubuntu:24.04 bash`

---

## What you will learn

- What users and groups are, and how Linux stores them (`/etc/passwd`, `/etc/group`, `/etc/shadow`)
- Create, inspect, modify, lock and delete users and groups
- Read and set **permissions** with `chmod` (symbolic and numeric) and change owners with `chown`
- Directories vs files: what `r`, `w`, `x` really mean
- `umask`, `sudo`, and the special bits (setuid, setgid, sticky)
- **Docker specifics:** why containers run as root by default, how to run as non-root, and how to fix permission errors with volumes

---

## 1. Users

Every process and every file belongs to a **user**. The kernel identifies users by number, the **UID**; names are just labels.

| Kind | UID | Purpose |
|---|---|---|
| **root** (superuser) | **0** | The administrator. The kernel skips most permission checks for UID 0 |
| **System users** | usually 1–999 | Accounts for services (`www-data`, `postgres`, `nobody`) that shouldn't log in |
| **Regular users** | usually 1000+ | People |

**Why not work as root?** A typo (`rm -rf /`), a malicious script, or a hacked program running as root can destroy or take over the whole system. The **principle of least privilege**: every program and person should have only the rights they need. That principle applies double inside containers (section 9).

### Inspect users

```bash
whoami                 # current user name
id                     # uid=0(root) gid=0(root) groups=0(root)
id alice               # info about another user
who                    # who is logged in (may be empty in a container)
cat /etc/passwd | head -5
```

### The user database: three files

| File | Holds | Readable by |
|---|---|---|
| `/etc/passwd` | Accounts: name, UID, GID, description, home, shell | Everyone |
| `/etc/shadow` | **Password hashes** and expiry settings | root only |
| `/etc/group` | Groups and their members | Everyone |

A line of `/etc/passwd` has **7 colon-separated fields**:

```
alice:x:1001:1001:Alice Smith:/home/alice:/bin/bash
│     │ │    │    │           │           └ login shell (/usr/sbin/nologin = cannot log in)
│     │ │    │    │           └ home directory
│     │ │    │    └ comment (full name)
│     │ │    └ primary group ID (GID)
│     │ └ user ID (UID)
│     └ "x" = the password hash is in /etc/shadow
└ user name
```

---

## 2. Managing users

You need root (or `sudo`) for these. In a container you already are root.

### Create

```bash
useradd -m -s /bin/bash alice     # -m create /home/alice, -s login shell
passwd alice                      # set a password (interactive)
id alice                          # uid=1001(alice) gid=1001(alice) groups=1001(alice)
ls -la /home/alice                # home directory was created (with -m)
```

- Without `-m`, `useradd` creates the account but **no home directory**.
- Without `-s`, Debian/Ubuntu's `useradd` gives a plain `/bin/sh`.
- **`adduser`** (Debian/Ubuntu only) is a friendlier wrapper: `adduser alice` asks questions and creates home, group and password. `useradd` is the portable low-level tool and is preferred in Dockerfiles and scripts.
- For non-interactive password setting in scripts, use `echo 'alice:secret' | chpasswd`. Never bake real passwords into images or scripts you share.
- For service accounts: `useradd --system --no-create-home --shell /usr/sbin/nologin appuser`.

### Switch users

```bash
su - alice          # start a login shell as alice ("-" loads her environment, goes to her home)
whoami
exit                # back to root
```

Root can `su` to anyone without a password; others must enter the target's password.

### Modify, lock, delete

```bash
usermod -s /bin/zsh alice         # change shell
usermod -L alice                  # LOCK the password (can't log in with it)
usermod -U alice                  # unlock
userdel alice                     # delete the account, KEEP home files
userdel -r alice                  # delete the account AND home directory and mail spool (permanent!)
```

Note: `usermod -L` locks the *password*. Other ways to log in (such as SSH keys) may still work; disabling the shell or expiring the account (`usermod --expiredate 1`) is stronger.

Passwords: a normal user changes their own with `passwd` (asks for the current one); root can set anyone's. Choose strong passwords in real life.

---

## 3. Groups

A **group** is a named set of users. Instead of granting a permission to ten people one by one, you grant it to a group and put the ten people in it.

Each user has:

- exactly one **primary group** (the `gid` in `id`). By default `useradd` creates a group with the same name as the user. New files the user creates are owned by this group.
- zero or more **supplementary (secondary) groups**.

```bash
id alice        # uid=1001(alice) gid=1001(alice) groups=1001(alice),1003(developers)
groups alice    # alice : alice developers
```

A line of `/etc/group` has 4 fields: `developers:x:1003:alice,bob` = name, password placeholder, GID, comma-separated members.

### Manage groups

```bash
groupadd developers                 # create
usermod -aG developers alice        # ADD alice to developers  (-a append, -G supplementary groups)
gpasswd -d alice developers         # remove alice from the group
groupdel developers                 # delete the group (not allowed while it is someone's primary group)
getent group developers             # look up a group (works with network directories too)
```

> ⚠️ **Always use `-aG` together.** `usermod -G developers alice` (without `-a`) **replaces** all of alice's supplementary groups with only `developers`, which can silently remove her `sudo` membership.

**Group changes apply to new logins.** A user who is already logged in must log out and back in (or run `newgrp developers`) for a new group to take effect.

---

## 4. Permissions

Every file and directory has:

- an **owner** (a user) and a **group**,
- three permission sets: for the **owner (u)**, for the **group (g)**, and for **others (o)**, everyone else,
- in each set, three switches: **read (r)**, **write (w)**, **execute (x)**.

```
 -  rw-  r--  r--   1  alice  developers   220 Jan 10 12:00  notes.txt
 │  │    │    │         │     │
 │  │    │    │         │     └ group owner
 │  │    │    │         └ owner
 │  │    │    └ others:  read only
 │  │    └ group:   read only
 │  └ owner:   read + write
 └ type: - file, d directory, l symlink
```

### What each letter means

| | On a **file** | On a **directory** |
|---|---|---|
| **r** (4) | View contents (`cat`) | **List** names inside (`ls`) |
| **w** (2) | Modify contents | **Create, delete or rename** entries inside (needs `x` too) |
| **x** (1) | **Run** it as a program or script | **Enter** it (`cd`) and access items inside by name |

Two consequences that surprise people:

1. To read `/a/b/c.txt` you need **`x` on every directory** on the path (`/a` and `/a/b`), plus `r` on the file.
2. **Deleting a file depends on the *directory's* `w`+`x`, not on the file's own permissions.** You can delete a read-only file if you can write to its directory.

### How Linux decides
When you access a file, the kernel checks **only one** of the three sets: if you're the **owner**, the owner set applies (even if it is more restrictive than the group's); else if you're in the **group**, the group set applies; else the **others** set. **Root** bypasses these read/write checks (though not "execute" on files that have no execute bit at all).

---

## 5. Changing permissions with `chmod`

### 5.1 Symbolic mode: `chmod [who][+-=][what] file`

| Who | Operator | What |
|---|---|---|
| `u` owner, `g` group, `o` others, `a` all | `+` add, `-` remove, `=` set exactly | `r`, `w`, `x` |

```bash
touch a.txt
ls -l a.txt                 # -rw-r--r--
chmod u+x a.txt             # owner may execute        → -rwxr--r--
chmod g+w a.txt             # group may write          → -rwxrw-r--
chmod o-r a.txt             # others lose read         → -rwxrw----
chmod a-x a.txt             # nobody may execute       → -rw-rw----
chmod u=rw,g=r,o= a.txt     # set exactly              → -rw-r-----
chmod -R g+rX project/      # recursive; capital X = add execute only on directories (and already-executable files)
```

### 5.2 Numeric (octal) mode
Add up the values `r=4`, `w=2`, `x=1` for each set:

| Digit | Bits | Meaning |
|---|---|---|
| 7 | 4+2+1 | rwx |
| 6 | 4+2 | rw- |
| 5 | 4+1 | r-x |
| 4 | 4 | r-- |
| 0 | 0 | --- |

Three digits = owner, group, others:

```
chmod 755 script.sh     # rwxr-xr-x   owner all; others read+run     (programs, directories)
chmod 644 notes.txt     # rw-r--r--   owner edits; others read       (normal files)
chmod 600 id_rsa        # rw-------   owner only                     (private keys, secrets)
chmod 700 private/      # rwx------   owner only                     (private directory)
chmod 664 shared.txt    # rw-rw-r--   owner+group edit
chmod 770 team_dir/     # rwxrwx---   owner+group only
chmod 444 readonly.txt  # r--r--r--   nobody writes (root still can!)
```

### 5.3 See the effect

```bash
echo hello > a.txt
chmod u-w a.txt
echo bye > a.txt         # bash: a.txt: Permission denied   (as a normal user)
chmod u+w a.txt
```

> ⚠️ **Root ignores read/write permission bits.** As root, the `echo bye > a.txt` above would **succeed** even on a `444` file. Permissions protect users from each other, not from root. (This is why running as root, in a container or not, is riskier.) To truly block writes even for root, file-system flags such as `chattr +i` exist.

### 5.4 Running a script

```bash
printf '#!/bin/sh\necho "Hello, World!"\n' > hello.sh
./hello.sh               # bash: ./hello.sh: Permission denied  (no x bit)
chmod +x hello.sh
./hello.sh               # Hello, World!
```

`sh hello.sh` also works without `x`, because you are running `sh`, and `sh` merely *reads* the file.

---

## 6. Changing owners with `chown` and `chgrp`

```bash
chown alice a.txt              # change owner
chown alice:developers a.txt   # owner and group
chown :developers a.txt        # group only  (same as: chgrp developers a.txt)
chown -R alice:alice /home/alice/project   # recursive
```

- Only **root** can give a file to a different user. If normal users could, they could dump their files onto someone else (for example to exhaust that user's disk quota or to make files look like someone else's).
- A regular user can change the **group** of their own file only to a group they belong to.

---

## 7. More concepts (intermediate)

### `umask`: default permissions for new files
New files start from `666` and new directories from `777`, minus the `umask`. The common umask is `022`, giving files **644** and directories **755**. (`umask` alone prints the current value; `umask 077` makes new files private for the rest of the session.)

### `sudo`: run one command as root

```bash
sudo apt update          # as a normal user: run this ONE command with root rights
sudo -l                  # what am I allowed to run?
sudo -u postgres psql    # run as another specific user
```

Who may use `sudo` is defined in `/etc/sudoers` (edit only with `visudo`) or by membership of the `sudo` group (Debian/Ubuntu) or `wheel` group (Red Hat family). Minimal Docker images usually don't include `sudo` because they run as root already.

### Special permission bits

| Bit | Numeric | On | Effect |
|---|---|---|---|
| **setuid** | 4xxx | executable | Runs with the **file owner's** privileges (e.g. `/usr/bin/passwd` runs as root so you can change your own password). Powerful, so a favourite target for attackers |
| **setgid** | 2xxx | directory | New files inherit the **directory's group**, ideal for shared team folders |
| **sticky** | 1xxx | directory | Only the file's owner (or root) may delete/rename files inside. `/tmp` is `drwxrwxrwt` |

```bash
ls -ld /tmp /usr/bin/passwd          # drwxrwxrwt ... /tmp    -rwsr-xr-x ... passwd
chmod 2770 /projects                 # setgid on a shared directory
find / -perm -4000 -type f 2>/dev/null   # audit: list setuid programs
```

### ACLs (just so you know)
When user/group/other isn't fine-grained enough, POSIX ACLs (`setfacl`, `getfacl`) can grant a specific extra user access to one file.

---

## 8. Worked scenarios

### 8.1 A shared team directory

```bash
groupadd developers
useradd -m -G developers alice
useradd -m -G developers bob
mkdir /projects
chown root:developers /projects
chmod 2770 /projects          # rwxrws---: team read/write; new files inherit the group; others: nothing
ls -ld /projects              # drwxrws--- 2 root developers ...

su - alice -c 'echo "from alice" > /projects/a.txt'
su - bob   -c 'cat /projects/a.txt'
su - nobody -s /bin/sh -c 'ls /projects'      # Permission denied
```

### 8.2 A private folder

```bash
su - alice
mkdir ~/private && chmod 700 ~/private
ls -ld ~/private              # drwx------
```

### 8.3 A protected config file (and its limits)

```bash
echo "SERVER=production" > /etc/app.conf
chown root:root /etc/app.conf
chmod 644 /etc/app.conf       # everyone reads, only root writes
```

---

## 9. Users, groups and permissions in Docker

### 9.1 Containers run as root by default
Unless the image says otherwise, `docker run` starts your process as **root (UID 0) inside the container**. Combined with the shared kernel (Chapter 4), that increases risk: if an attacker escapes the container, or if a bind-mounted host path is writable, root in the container can do real damage. **Best practice: run as a non-root user.**

### 9.2 Non-root in a Dockerfile

```dockerfile
FROM ubuntu:24.04

# create an unprivileged user and group with fixed IDs
RUN groupadd --gid 10001 app \
 && useradd  --uid 10001 --gid app --create-home --shell /usr/sbin/nologin app

WORKDIR /app
COPY --chown=app:app . /app     # files owned by the app user

USER app                         # every later instruction and the container's process run as "app"
CMD ["./run.sh"]
```

Notes:
- Use **numeric UIDs** (`USER 10001:10001`); orchestrators such as Kubernetes can then verify "non-root".
- On Alpine the commands are `addgroup -g 10001 app` and `adduser -D -u 10001 -G app app`.
- Many official images already provide a user (`nginx`, `postgres`, `node`). For example, `node` images include a user named `node`: `USER node`.
- Non-root users **cannot bind ports below 1024** by default; use a high port (8080) and map it (`-p 80:8080`).

### 9.3 Choosing the user at run time

```bash
docker run --rm ubuntu:24.04 id                       # uid=0(root)
docker run --rm --user 1000:1000 ubuntu:24.04 id      # uid=1000 gid=1000 (no name: not in /etc/passwd, and that's fine)
docker run --rm -u nobody ubuntu:24.04 id
docker exec -u root -it mycontainer bash              # a root shell in a running container for debugging
```

### 9.4 The big trap: UIDs are numbers shared with the host
Containers share the host kernel, and **the kernel only knows numbers**. The name "alice" inside a container means nothing to the host; what matters is the UID.

```bash
# on the host, as user with UID 1000:
mkdir data
docker run --rm -v "$PWD/data:/data" ubuntu:24.04 sh -c 'touch /data/from_container'
ls -ln data           # from_container is owned by UID 0 on the host (root!)
```

Consequences and fixes for **bind mounts**:

| Problem | Cause | Fix |
|---|---|---|
| Files created by the container are owned by root on the host | Container ran as root | Run with `--user "$(id -u):$(id -g)"` |
| `Permission denied` writing to a mounted folder | Container user's UID differs from the folder owner's | Match UIDs (`--user`), or `chown` the folder to the UID the container uses, or use a named volume |
| Works on Linux, fails on Docker Desktop (or vice versa) | Docker Desktop's file sharing translates ownership differently | Test on your target platform; prefer named volumes for data |
| SELinux systems (Fedora/RHEL): `Permission denied` even with right UID | SELinux labels | Add `:z` or `:Z` to the mount: `-v ./data:/data:z` |

**Named volumes** are managed by Docker and are a good default for databases, and Docker copies the image's ownership of the mount point into a new empty volume the first time.

### 9.5 More hardening (expert)

```bash
docker run --rm \
  --user 10001:10001 \        # non-root
  --read-only \               # read-only root file system
  --tmpfs /tmp \              # writable scratch space
  --cap-drop ALL \            # drop all Linux capabilities (add back only what is needed)
  --security-opt no-new-privileges \   # block setuid privilege gain
  myimage
```

Other options: **user namespaces** (`userns-remap`) map container root to an unprivileged host UID; **rootless Docker/Podman** runs the whole engine unprivileged; **seccomp/AppArmor** profiles limit system calls. Never use `--privileged` unless you fully understand why.

---

## 10. Common errors

| Symptom | Cause | Fix |
|---|---|---|
| `Permission denied` running `./script.sh` | No execute bit | `chmod +x script.sh` |
| `bash: ./script.sh: /bin/bash^M: bad interpreter` | Windows line endings (CRLF) | `dos2unix`, or set `.gitattributes` to LF |
| `cat: file: Permission denied` | No `r` for your class, or no `x` on a directory in the path | `ls -ld` each directory in the path; `namei -l /path/to/file` shows all permissions along the path |
| `chown: Operation not permitted` | Not root | Use `sudo`, or `--chown` in Docker `COPY`/`ADD` |
| `sudo: command not found` | Minimal image | You're probably root already; else `apt-get install sudo` |
| User can't use newly-added group | Session started before the change | Log out/in or `newgrp` |
| `groupdel: cannot remove the primary group of user` | The group is someone's primary | Delete or change that user first |
| Container's files owned by root on the host | Running as root with bind mounts | `--user "$(id -u):$(id -g)"` |
| Web server can't read uploaded files | Wrong owner/group, or missing `x` on a parent directory | Fix ownership (`chown www-data`) and directory `x` bits |

---

## 11. Best practices

1. Work as a **normal user**; use `sudo` for specific tasks.
2. **Least privilege**: give the minimum access needed.
3. Avoid **`chmod 777`**. It hides the real problem and lets anyone modify the file. Find the correct owner/group instead.
4. Use **groups** for shared access, not world-writable files.
5. Private data: files `600`, directories `700` (SSH private keys **must** be `600` or SSH refuses them).
6. **Containers:** run as non-root, use numeric IDs, use `COPY --chown`, consider `--read-only` and `--cap-drop ALL`.
7. Audit occasionally: `find / -xdev -perm -0002 -type f` (world-writable files), `find / -xdev -perm -4000` (setuid programs).

---

## 12. Cheat sheet

| Task | Command |
|---|---|
| Who am I / IDs / groups | `whoami` · `id` · `groups` |
| Create user (home, shell) | `useradd -m -s /bin/bash NAME` |
| Set password | `passwd NAME` |
| Switch user | `su - NAME` |
| Lock / unlock | `usermod -L NAME` · `usermod -U NAME` |
| Delete (with home) | `userdel -r NAME` |
| Create group / add user / remove user | `groupadd G` · `usermod -aG G NAME` · `gpasswd -d NAME G` |
| Show permissions | `ls -l` · `ls -ld dir` · `stat file` |
| Change permissions | `chmod u+x f` · `chmod 644 f` · `chmod -R g+rX d` |
| Change owner/group | `chown user:group f` · `chown -R ...` |
| Run as root | `sudo CMD` |
| Docker: run as user | `docker run --user 1000:1000 ...` · `USER app` |

---

## 13. Check your understanding

1. What does `-rwxr-x---` mean for the owner, the group and others?
2. What is the numeric equivalent of `rw-r-----`?
3. Why does `usermod -G` (without `-a`) risk removing someone's access?
4. A file is `chmod 444`. Can root still overwrite it? Can a normal file owner?
5. You can `ls` a directory but cannot `cd` into it. Which permission is missing?
6. A file has permission `r--rw-rw-` and you are its owner. Can you write to it? Explain.
7. Your container writes files into a bind-mounted directory, and they are owned by root on the host. Give two ways to prevent it.

<details>
<summary>Answers</summary>

1. Owner: read, write, execute. Group: read and execute. Others: nothing.
2. 640
3. It *replaces* the user's supplementary groups with only the listed ones, dropping others such as `sudo`. Use `-aG`.
4. Root: yes (permission bits don't stop root). A normal owner: no, unless they first `chmod u+w`.
5. Execute (`x`) on the directory.
6. No. Linux applies only the **owner** class to the owner, which is `r--` here, even though the group and others could write. (The owner could `chmod` it first.)
7. Run the container as your own UID (`--user "$(id -u):$(id -g)"`, or a `USER` in the image), or chown the directory to the UID the container runs as, or use a named volume.
</details>

**Practice:** create user `alice` and `bob` in a container, put both in group `team`, create `/shared` with mode `2770` owned by `root:team`, and prove that alice's files are readable and writable by bob but not by a third user.

---

**Next:** [Chapter 14 – Docker Hands-On](14_docker_hands_on.md)
