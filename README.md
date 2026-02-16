# public-backup-grep (C)

A small C utility that **recursively scans multiple site directories** under a given root path and reports potentially sensitive files that appear to be **publicly accessible** (e.g., exposed database dumps or WordPress config backups).

It’s designed for hosting-style layouts where sites typically live two directory levels below a base directory, and where web roots include a `/public/` folder.

---

## What it does

Given a `<path>`, the program:

1. **Traverses directories up to depth 2** (collecting “site roots” at depth 2).
2. For each collected site root, it runs a recursive scan (`grep()`).
3. It prints matches only when:
   - The file name looks like a backup/dump:
     - `wp-config` with `.bac`, `.bak`, or `.backup`
     - any `*.sql`
   - The file path contains `/public/`
   - The file appears **publicly accessible**, based on filesystem permissions and (optionally) ACLs.

Output includes a timestamp and a per-site thread id (TID).

Example output:
```[2026-02-16 12:34:56] [TID:0007] Found: /srv/sites/example/public/wp-config.php.bak```

---

## How “publicly accessible” is determined

A file is considered publicly accessible if:

- The file is **world-readable** (`S_IROTH`), AND
- All parent directories up to (and including) a directory containing `/public` are traversable.

Special handling for the `/public` directory:
- If `/public` has **world-execute** (`S_IXOTH`), it’s considered accessible.
- Otherwise it checks for an **ACL execute permission** for UID `33` (defaults to `www-data` on Debian/Ubuntu):
  - `WWW_DATA_UID` is set to `33`
  - `has_www_data_exec_acl()` inspects POSIX ACL entries for `ACL_USER` matching that UID and requiring `ACL_EXECUTE`

> Note: this logic is intentionally conservative. It attempts to approximate “can a web server user reach this file?”.

---

## Directory exclusions

During traversal (site discovery), these directory names are skipped:

- `.`, `..`
- `backup`
- `phpmyadmin`
- `defaultsite`
- `lost+found`

(See `dont_want_dir()`.)

---

## Build

This program uses:
- pthreads (`-pthread`)
- POSIX semaphores
- libacl (`-lacl`)

On Debian/Ubuntu, you’ll typically need:
- `build-essential`
- `libacl1-dev`

Build:
```bash
gcc -O2 -Wall -Wextra -pedantic -pthread main.c -lacl -o public-backup-grep

