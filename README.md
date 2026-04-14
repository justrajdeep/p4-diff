# p4-diff

`p4-diff` is a small Python CLI that prints a **unified diff** for every file in a Helix Core / Perforce **changelist**. It supports **pending (opened) changelists** and **shelved** changelists, and handles **add**, **edit**, **integrate**, **delete**, and related move/branch operations.

The script shells out to `p4` and to your system `diff` (or whatever `P4DIFF` points at for Perforce-invoked paths). It does **not** use the Perforce C++ API.

## Requirements

- `p4` on your `PATH` (the script runs `which p4` before doing anything).
- Python **3** via **`uv`** (first line is `#!/usr/bin/env -S uv run python3`). Run with `./p4-diff` after `chmod +x`, or `uv run python3 p4-diff …`.

## Usage

```text
p4-diff [--changelist CL] [-s] [-l] [-t TICKET] [-v]
```

| Option | Meaning |
|--------|--------|
| `-c` / `--changelist` | Changelist number; **default is `default`** (your default pending changelist) when omitted. Use an explicit number for shelved CLs. |
| `-s` / `--shelved` | Treat the CL as **shelved** (pending files on the server) instead of **opened** in your client. |
| `-l` / `--local` | **Only with `-s`:** for each file, write shelved content with **`p4 print`** to a temp file, then **`diff -u <workspace-or-null> <temp>`**. Workspace path comes from **`p4 where`**; if the file is **missing** on disk, the left side is **`/dev/null`**. |
| `-t` / `--ticket` | Passes **`-P TICKET`** to every `p4` invocation (ticket-based auth). |
| `-v` / `--verbose` | Prints each command before it runs. **Warning:** this output is mixed with real diff text and will break consumers that expect a clean patch stream. |

```bash
./p4-diff --help
```

## What it does (two modes)

### 1. Opened changelist (no `-s`)

File list comes from:

```text
p4 opened -s -c <CL>
```

For each depot path it maps to a workspace file with `p4 where` when it needs a local path.

| Operation | Behavior |
|-----------|----------|
| **add**, **branch**, **move/add** | Runs **`diff -u /dev/null <local-file>`** so the patch is “new file” vs empty. |
| **edit**, **integrate** | Runs **`p4 diff -du <depot-path>`**. That is Perforce’s normal **client workspace vs opened revision** diff (your checked-out content is involved). |
| **delete**, **move/delete** | Runs **`diff -u <local-file> /dev/null`** (file disappearing). |

“Empty” detection for edits: if `p4 diff` returns only **three** lines (header-only), the script treats it as no diff and prints nothing for that file (unless `-v`).

### 2. Shelved changelist (`-s`)

File list comes from:

```text
p4 describe -S -s <CL>
```

It parses only lines starting with `...` and expects the word layout this script was written for (see **Limitations**).

| Operation | Behavior |
|-----------|----------|
| **add**, **branch**, **move/add** | **`p4 fstat`** (head type) and **`p4 print -q depot@=CL`** into a temp file, then **`diff -u /dev/null temp`**. The second diff header line is rewritten to show the **depot** path instead of the temp path. Symlinks: strips an extra trailing newline Perforce adds. |
| **edit**, **integrate** | **`p4 diff2 -u depot#<baseRev> depot@=<CL>`** — compares the **depot revision before the shelf** with the **shelved depot revision**. This is **not** “shelved server file vs my local disk”; both sides are **depot** revisions. |
| **delete**, **move/delete** | **`p4 print`** of the **pre-delete depot revision** (`#rev` from describe), temp file, then **`diff -u temp /dev/null`**, with the first header line rewritten to the depot path. Same symlink newline tweak as adds. |

“Empty” detection for shelved edits: if `diff2` output splits to a **single** line, treated as empty (unless `-v`).

### 3. Shelved changelist + **`-l` / `--local`** (`-s -l`)

Requires **`-s`**. For every shelved file, the script:

1. Materializes the **shelved** revision with **`p4 print -q`** into a **temp file** (same symlink newline handling as the default shelved-add path). Revision spec is **`depot@=CL`** for all operations except **`delete` / `move/delete`**, which use **`depot#rev`** from `p4 describe` (the revision being removed).
2. Resolves the workspace path with **`p4 where`**.
3. Runs **`diff -u LEFT TEMP`** where **`LEFT`** is the local file if **`os.path.isfile`** is true, else **`/dev/null`**.
4. Rewrites diff headers so the left side is labeled **`… (workspace)`** when present and the right side **`… (shelved)`**.

This is the mode to use when you care about **your tree vs the shelf**, not **`p4 diff2`** between two depot revisions.

| Operation | `p4 print` spec | Left side of `diff -u` |
|-----------|-----------------|-------------------------|
| **add**, **branch**, **move/add**, **edit**, **integrate** | `depot@=CL` | Workspace file or `/dev/null` |
| **delete**, **move/delete** | `depot#rev` | Workspace file or `/dev/null` |

**Note:** This is **similar in spirit** to **`p4 diff file@=CL`** for edits, but implemented as **`diff`** on **`p4 print`** output vs the file on disk, so whitespace or encoding handling may not match Perforce’s internal diff exactly.

## Environment: `P4DIFF`

Before each `p4` subprocess, the script copies your environment and sets **`P4DIFF`** to `diff` unless **`P4DIFF`** is already set. That controls which program `p4` uses when it needs an external diff. Your plain `diff -u` invocations for add/delete still use the string `diff` from `get_diff_tool()` (same default / same override).

## Shelved vs local workspace

- **Default shelved mode (`-s` only):** **edit**/**integrate** use **`p4 diff2`** (depot `#base` vs depot `@=shelf`), not your workspace file.
- **`-s -l`:** workspace file (or **`/dev/null`**) vs **`p4 print`** of the shelved revision — see **§3** above.

For a single file, native **`p4 diff path@=<shelfCL>`** may still differ in details from **`-l`** (Perforce vs external **`diff`**).

## Examples

```bash
# Pending change: diff everything in your default changelist (same with or without -c default)
./p4-diff > default.diff
./p4-diff -c default > default.diff

# Shelved change 123456 (depot-vs-depot for edits)
./p4-diff -c 123456 -s > shelved_123456.diff

# Shelved change: workspace (or /dev/null) vs shelved print for every file
./p4-diff -c 123456 -s -l > shelved_123456_vs_local.diff

# Opened (non-shelved) numbered change
./p4-diff -c 123456 > opened_123456.diff
```

Dry-run applying a saved patch (strip level depends on how the diff was generated; adjust `-p` as needed):

```bash
cd ~/other_tree
patch --dry-run -Np1 -i opened_123456.diff
```

## Output and exit behavior

- Diffs are written to **stdout**; errors from `p4` on stderr cause the script to print them and **`sys.exit(1)`**.
- Subprocess text is decoded as **UTF-8** with **`errors='replace'`**; binary-ish depot files may not round-trip perfectly through the temp-file path.

## Limitations / caveats

1. **`p4 describe -S -s` parsing** is line-based and assumes fields split on spaces match the script’s expectations; unusual depot paths or future output changes can break parsing.
2. **`p4 where` output** is split on spaces and the **third** token is taken as the local path — same fragility for paths with spaces.
3. **Temp files** for shelved add/delete use `NamedTemporaryFile(delete=False)`; they are not deleted by the script (same as the original design intent: short-lived paths for `diff`).
4. **`integrate`** is grouped with **edit** for diff generation.
5. **`--verbose`** corrupts a pure-patch stdout stream by interleaving command traces.
6. **Omitting `-c`** uses the literal changelist name **`default`**, which is right for **`p4 opened -c default`** but is often **wrong for shelved work** (`p4 describe -S`); pass **`-c <number>`** for shelves.

## Repository layout

| File | Role |
|------|------|
| `p4-diff` | Executable script (Python 3, `uv` shebang). |
| `README.md` | This document. |

Related public tool with the same general idea: [JonParr/p4-diff](https://github.com/JonParr/p4-diff). This tree may include local changes (Python 3, `uv` shebang, README).
