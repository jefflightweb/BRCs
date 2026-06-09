---
name: brcs
description: Triage and merge open PRs on the bitcoin-sv/BRCs repository. Assigns unique sequential BRC numbers, renames files when needed, wires links into README.md / SUMMARY.md / dir README. Use when user says "/brcs", "merge BRC PRs", "triage BRCs", "process BRC pull requests", or wants to drain the open PR queue on the BRCs docs repo.
user-invocable: true
allowed-tools:
  - Bash
  - Read
  - Edit
  - Write
  - AskUserQuestion
  - TaskCreate
  - TaskUpdate
  - TaskList
---

# /brcs — Triage and Merge BRC PRs

Drain open PRs on bitcoin-sv/BRCs. Assign next BRC numbers. Wire indices. Push.

## Preconditions

- `cwd` is a clone of `bitcoin-sv/BRCs` (check `git remote -v`).
- `gh` authed, push access to `origin/master`.
- On `master`, clean (or only `.DS_Store` untracked).

If preconditions fail → tell user, stop.

## Pipeline

```
list PRs → read README index for highest BRC# → triage each PR →
  (close duplicates | merge content updates | merge new BRCs with renumber if needed) →
  resolve index conflicts → push → report
```

## Step 1 — Snapshot

```bash
gh pr list --state open --limit 50
```

Read `README.md` "## Standards" table. Highest existing `NNN | [Title](path)` row = `LAST_BRC`. Next available = `LAST_BRC + 1`.

Also note structure:
- Per-category dir READMEs: `apps/README.md`, `wallet/README.md`, `transactions/README.md`, `payments/README.md`, `overlays/README.md`, `peer-to-peer/README.md`, `key-derivation/README.md`, `outpoints/README.md`, `opinions/README.md`, `tokens/README.md`, `scripts/README.md`, `state-machines/README.md`.
- `SUMMARY.md` (GitBook index) groups by category heading.
- Top-level `README.md` has flat `BRC | Standard` table.

## Step 2 — Triage each PR

For each open PR (ascending creation date), inspect:

```bash
gh pr view <N> --json number,title,state,mergeable,files,headRefName,baseRefName,isDraft
gh pr diff <N> --patch | head -60
git fetch origin pull/<N>/head:pr-<N>
```

Classify into one of:

### (a) DRAFT
Skip. Mention in final report.

### (b) Duplicate — content already in master
Symptoms:
- Adds `XXX.md` where same-named file already exists with byte-identical content (modulo BRC number references), OR
- Modifies existing file but `diff <(git show pr-<N>:path) path` is trivial (whitespace/formatting only).

Action: confirm with user; on approval, `gh pr close <N> --comment "Closing — content already merged to master as BRC-YYY (path/, commit ZZZZZ). ..."`.

### (c) Content update to existing BRC
PR's latest commit modifies an existing BRC file with non-trivial changes (rework, clarifications, field renames). No new BRC number needed.

Action: `git merge --no-ff --no-commit pr-<N>`. Resolve conflicts (see Step 4). Also update any title-bearing references in `SUMMARY.md` + dir README if the BRC's title changed. Commit with body referencing PR.

### (d) New BRC
PR adds a new `<dir>/NNNN.md`. Inspect proposed number vs. `LAST_BRC + 1`.

- **If proposed number matches next slot AND no PR earlier in queue claims it** → keep number, just merge.
- **If proposed number is taken or queue order conflicts** → reassign:
  1. Compute `ASSIGNED = LAST_BRC + 1`.
  2. `git mv <dir>/<PROPOSED>.md <dir>/<ASSIGNED>.md` in the PR's branch (or apply rename after merge).
  3. In the renamed doc, `sed -i '' "s/BRC-<PROPOSED>/BRC-<ASSIGNED>/g"` (or Edit) — but **only the document's self-references** (title heading + any "this BRC" / "BRC-XXX" pointers pointing at itself). Do NOT change references to *other* BRCs.
  4. Track the mapping `PROPOSED → ASSIGNED` for this session.
  5. Before merging each subsequent PR, grep its files for any reference to a `PROPOSED` number that has since been remapped and rewrite to `ASSIGNED`.

Action: merge, resolve conflicts, update indices (Step 5), commit. Increment `LAST_BRC`.

## Step 3 — Track BRC number mapping

Maintain a session map (mental or scratch file) of `PROPOSED → ASSIGNED` for every PR processed in this run. Other PRs in the queue may cross-reference these numbers; rewrite references when you encounter them.

Check cross-references:

```bash
git show pr-<N>:<file> | grep -oE "BRC-[0-9]+" | sort -u
```

For each match, if the number appears in the session map under `PROPOSED`, plan to rewrite the reference to `ASSIGNED` before commit.

## Step 4 — Resolve index conflicts

PRs typically modify the same lines in `README.md`, `SUMMARY.md`, and the dir's `README.md`. After `git merge --no-ff --no-commit`, conflicts appear in those files.

For the common case (each PR appends a single new line, conflict between HEAD's growing list and PR's single line):

```bash
sed -i '' '/^<<<<<<< HEAD$/d; /^=======$/d; /^>>>>>>> pr-/d' README.md SUMMARY.md <dir>/README.md
```

This strips the markers, keeping HEAD's lines followed by PR's new line — correct order.

Verify:

```bash
grep -n "<<<<<<<\|>>>>>>>\|=======" README.md SUMMARY.md <dir>/README.md || echo clean
```

If any PR pulled in unrelated stale changes (e.g. removed sections because its branch base predates a refactor on master), `git checkout HEAD -- <file>` and re-add the intended one-line addition manually.

## Step 5 — Wire indices

Every new BRC needs entries in **three** places (four if there's a dir README beyond the standard set):

1. **`README.md`** top-level table — append row to the `BRC | Standard` table just before `## License`:
   ```
   NNN  | [Title](./<dir>/NNNN.md)
   ```

2. **`<dir>/README.md`** category table — append to its `BRC | Standard` table:
   ```
   NNN  | [Title](./NNNN.md)
   ```

3. **`SUMMARY.md`** — append under the matching `## <Category>` section:
   ```
   * [Title](./<dir>/NNNN.md)
   ```

PR authors usually update some but not all. Diff PR's `files` list against this 3-place requirement and add the missing entries manually before commit.

## Step 6 — Commit

```bash
git commit -m "$(cat <<'EOF'
feat: add BRC-NNN <Title> (#<PR>)

Merges PR #<PR> from <head_ref>.

Co-authored-by: <Author Name> <author@example.com>
EOF
)"
```

For content updates use `refactor:` or `docs:` instead. Always reference the PR number.

## Step 7 — Push and report

After all PRs processed:

```bash
git log --oneline master ^origin/master
```

Confirm list with user. Push:

```bash
git push origin master
```

GitHub auto-detects merged commits and closes the merged PRs. Verify:

```bash
gh pr list --state open --limit 50
```

Report final mapping table to user:

```
Closed (duplicates): #X, #Y
Merged (content updates): #A → BRC-NNN, ...
Merged (new BRCs): #P → BRC-MMM, ...
Remaining open: #Q (reason: draft / blocked / ...)
```

## Guardrails

- **Never** modify the substance of a contributor's document beyond:
  - BRC number reassignment (title heading + self-references only)
  - Cross-reference rewrites for renumbered peers (Step 3)
  - Whitespace/conflict-marker cleanup
- **Always** confirm with user before closing a PR — closing is an external write under the user's identity.
- **Always** verify duplicate claim by diffing content, not just by title or BRC number.
- Pushing direct to `master` bypasses branch protection — user has authorised this for the docs repo. If push is rejected with a hard block, stop and ask.
- Use `TaskCreate` per PR to keep status visible; update as you go.
- If a PR's branch base predates a recent master refactor, the merge may pull in stale reverts. Inspect `git diff --cached --stat` before committing every merge — restore from `HEAD` any unrelated changes.
