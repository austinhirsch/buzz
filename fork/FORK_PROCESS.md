# Fork Process Guide — Customizing Buzz While Tracking Upstream

This fork (`austinhirsch/buzz`) customizes [block/buzz](https://github.com/block/buzz)
while continuing to take upstream changes. This guide covers where changes go,
how to make them, and how to sync.

**The goal: keep the diff against upstream small, easy to find, and in files
upstream rarely touches.** Every line changed in an upstream file can conflict on
a later sync.

---

## 1. Why this needs a process

Upstream moves quickly. Measured on 2026-09-14:

| Metric | Value |
|---|---|
| Upstream commits, last 30 days | **329** (~11/day) |
| Hottest paths (file touches, 30 days) | `desktop/src` 2184, `desktop/src-tauri` 777, `mobile/lib` 445, `desktop/tests` 270, `mobile/test` 188 |
| Cooler paths | `crates/buzz-db` 146, `crates/buzz-relay` 132, `crates/buzz-acp` 91, `crates/buzz-cli` 41 |

Edits to `desktop/src` will conflict most often. Choose where to put
customizations with that in mind.

---

## 2. Branch and remote model

```
upstream  → https://github.com/block/buzz.git        (read-only, source of truth)
origin    → https://github.com/austinhirsch/buzz.git (our fork)
```

| Branch | Purpose | Rules |
|---|---|---|
| `main` | Our trunk: upstream plus our customizations | Change it only by PR. Never commit or push to it directly. |
| `fork/<topic>` | A customization in progress | Branch from `main`, then PR into `main` |
| `sync/upstream-YYYY-MM-DD` | One upstream merge | Branch from `main`, merge `upstream/main` into it, then PR into `main` |
| `upstream-pr/<topic>` | A change we want to contribute to upstream | Branch from **`upstream/main`**, not our `main` |

**Merge upstream in. Do not rebase onto it.** With about 330 upstream commits a
month, rebasing a stack of fork commits means resolving the same conflicts again
every time, and it rewrites history that our PRs already reference. Merging
resolves each conflict once, and `git rerere` (see §5) remembers the resolution.

### One-time setup

```bash
git remote add upstream https://github.com/block/buzz.git   # already done
git config rerere.enabled true                               # remember conflict resolutions
git config rerere.autoupdate true
gh repo set-default austinhirsch/buzz                        # IMPORTANT — see below
```

> ⚠️ **`gh` targets the parent repo by default in a fork.** Without
> `gh repo set-default`, `gh pr create` opens the PR against **block/buzz**.
> That would publish our customizations to Block. Check the target before every
> `gh pr create`, or pass `--repo austinhirsch/buzz` explicitly.

### GitHub Actions on the fork

Upstream has 26 workflows. Many need Block's secrets or infrastructure (release
signing, ECR pushes, Helm charts, canaries). On the fork:

- In the fork's **Settings → Actions**, allow only `ci.yml` and the `_ci-*.yml`
  reusable workflows it calls. Disable release, docker, helm, canary, and
  `codex-security-review`.
- Do not edit upstream workflow files to disable them. Edits would conflict on
  every sync. Use repository settings instead.

---

## 3. Customization tiers: choose the lowest tier that works

Each tier costs more to maintain than the one above it. Before you build,
place the change in a tier and write down that choice in the PR.

### Tier 0: Configuration (no code, no conflicts)

- `.env` (copied from `.env.example`, which has 99 keys). `.env` is gitignored,
  so it never conflicts.
- Relay administration through `buzz-admin`
- Runtime settings in the app: communities, themes, font size, notifications
- Deployment values that live **outside** this repo

If a value is hard-coded upstream and you want it configurable, consider
submitting a PR upstream that makes it configurable (Tier 5). A PR like that
usually gets accepted, and then the fork diff for that value is zero.

### Tier 1: Content through existing extension points (almost no conflicts)

Upstream built these extension points for this kind of customization:

| Extension point | Where | Spec |
|---|---|---|
| **Persona packs** (agent identity, system prompt, skills, MCP servers, hooks) | Keep them in `fork/personas/` or a separate repo | `crates/buzz-persona/PERSONA_PACK_SPEC.md` (a superset of the Open Plugin Spec) |
| **Workflows** (YAML automations with evalexpr conditions) | `fork/workflows/` | `crates/buzz-workflow` |
| **Agent skills** | `fork/skills/` | Examples in `examples/meadow-core/skills` |
| **Channel templates, custom emoji** | Created in the app; stored as relay events | `desktop/src/features/channel-templates`, `custom-emoji` |

Store this content under `fork/` or in a separate repository. **Do not add
files to upstream's `.claude/skills`, `.agents/skills`, or
`benchmarks/…/personas` directories.** Upstream changes those directories.

### Tier 2: Additive code in fork-owned namespaces (low conflict)

New files never conflict. If you need code, put almost all of it in **new
files** that upstream will never create:

| Surface | Fork namespace |
|---|---|
| Rust crates | `crates/fork-<name>/` (add to workspace `members`, which is a one-line change) |
| Desktop features | `desktop/src/features/fork-<name>/` |
| Tauri commands | `desktop/src-tauri/src/fork/` |
| Mobile features | `mobile/lib/features/fork_<name>/` |
| Event kinds | A reserved integer range; see §4.2 |
| Scripts / tooling | `fork/scripts/` |
| Docs | `fork/` |

Then connect the new code to upstream code with the **fewest possible hook lines**:
one import plus one registration call, one route entry, or one workspace
member. Those hook lines are the only part that can conflict, and they are easy
to re-apply when they do.

### Tier 3: Edits to upstream files (conflicts: minimize and mark)

Sometimes you cannot avoid changing an upstream file. When you must:

1. **Keep the edit as small as possible.** Change the fewest lines you can. Put
   the logic in a Tier 2 file and call it from the edited line.
2. **Mark every edited line or block** so that a single grep finds all of them:
   ```rust
   // FORK(austinhirsch): <why> — see fork/CUSTOMIZATIONS.md#<id>
   ```
   ```ts
   // FORK(austinhirsch): <why> — see fork/CUSTOMIZATIONS.md#<id>
   ```
   ```dart
   // FORK(austinhirsch): <why> — see fork/CUSTOMIZATIONS.md#<id>
   ```
3. **Record the edit in [`CUSTOMIZATIONS.md`](CUSTOMIZATIONS.md).**
4. **Do not reformat, reorder imports, or make unrelated cleanups** in upstream
   files. Whitespace changes cause conflicts too. The pre-commit hooks run
   formatters, so check `git diff` for incidental changes before you commit.
5. **Never edit `AGENTS.md` / `CLAUDE.md`.** `CLAUDE.md` is a symlink to
   `AGENTS.md`, and upstream edits `AGENTS.md` often. Put fork-specific agent
   guidance in `fork/AGENTS.fork.md` and load it from your personal
   `~/.claude/CLAUDE.md` or a `CLAUDE.local.md` (gitignored).

### Tier 4: Branding and identity (deliberate, planned divergence)

Ship branding changes **only if** we distribute our own builds:

| What | File | Current |
|---|---|---|
| Desktop product name | `desktop/src-tauri/tauri.conf.json` → `productName` | `Buzz` |
| Desktop bundle ID | `desktop/src-tauri/tauri.conf.json` → `identifier` | `xyz.block.buzz.app` |
| Mobile package | `mobile/pubspec.yaml`, iOS/Android bundle IDs | `buzz` |
| Theme | `desktop/src/shared/theme`, `mobile/lib/shared/theme`, `web/src/shared/theme` | Catppuccin |

Change the bundle identifier **before** anyone installs a fork build. If you
change it afterward, macOS and iOS treat the build as a different app, and local
data does not carry over. Also, do not reuse `xyz.block.*` for builds we
distribute.

### Tier 5: Upstream it (eliminates the diff)

If a customization would help anyone running Buzz, contribute it to
`block/buzz` instead. Examples: a bug fix, a configuration option for a
hard-coded value, a new extension point. After upstream merges it, delete our
copy. §6 covers the process.

---

## 4. Rules for high-risk areas

### 4.1 Database migrations

Upstream migrations use sequential numbers (`migrations/0044_…sql` as of this
writing) and run automatically on relay startup. **A fork migration numbered
`0045` will collide with upstream's next migration.**

- Avoid schema changes if you can. Store data as Nostr events (Tier 2), as
  AGENTS.md recommends.
- If you cannot avoid a migration, discuss it before building. There is no safe
  default. Options include a separate schema or database owned by a fork crate,
  or contributing the migration to upstream.
- `schema/schema.sql` and `scripts/reconcile-schema-after-pgschema.sql` are
  upstream-owned. Edits to them are Tier 3.

### 4.2 Event kinds

Kinds are defined in `crates/buzz-core/src/kind.rs`, and the desktop
(`desktop/src/shared/constants/kinds.ts`) and mobile
(`lib/shared/relay/nostr_models.dart`) copies must stay in sync.

- Before you add a kind, pick an integer range that upstream and the Nostr NIPs
  do not use, and record it in `CUSTOMIZATIONS.md`.
- Define fork kinds in a Tier 2 file and register them with a single hook line.
- After each sync, confirm upstream has not claimed a number we use:
  ```bash
  git diff <last-sync>..upstream/main -- crates/buzz-core/src/kind.rs
  ```

### 4.3 Lockfiles

Never resolve a conflict in `Cargo.lock` or `pnpm-lock.yaml` by hand. Take
upstream's version, then regenerate:

```bash
git checkout --theirs Cargo.lock pnpm-lock.yaml
cargo check --workspace   # updates only what changed; never `cargo generate-lockfile` (bumps every dep)
pnpm install
```

`--theirs` refers to the branch being merged in, which is upstream.

### 4.4 Community-scoped state (desktop)

If a Tier 2 desktop feature adds a module-level cache, Map, or singleton, it must
register a reset in `resetCommunityState()` in
`desktop/src/features/communities/useCommunityInit.ts`. That registration is a
Tier 3 hook line: mark it and record it in the ledger.

---

## 5. Syncing upstream

**Cadence: weekly, and before starting any large customization.** At about 11
commits a day, a week is roughly 75 commits. A month is roughly 330, and
conflicts become much harder to resolve.

### Procedure

```bash
. ./bin/activate-hermit                     # pinned toolchain for hooks
git switch main && git pull origin main
git fetch upstream

# 1. See what's coming and where it overlaps us
git log --oneline main..upstream/main | wc -l
git diff --stat main...upstream/main | tail -1
grep -rn "FORK(austinhirsch)" --include='*.rs' --include='*.ts' --include='*.tsx' \
  --include='*.dart' --include='*.json' --include='*.toml' . | cut -d: -f1 | sort -u \
  > /tmp/fork-touched.txt
git diff --name-only main...upstream/main | grep -Fxf /tmp/fork-touched.txt   # likely conflicts

# 2. Merge on a sync branch
git switch -c sync/upstream-$(date +%F)
git merge upstream/main --signoff -m "Merge upstream block/buzz $(git rev-parse --short upstream/main)"

# 3. Resolve conflicts (rerere replays prior resolutions)
git status
#    - lockfiles: see §4.3
#    - FORK-marked blocks: keep upstream's new code, re-apply our minimal hook
#    - anything not FORK-marked conflicting = an unrecorded customization; add to ledger
git add -A && git commit -s --no-edit

# 4. Verify
just setup          # deps + migrations may have changed upstream
just ci             # full gate
just test           # if relay/db/auth changed upstream or we touch them

# 5. Confirm fork customizations survived
grep -rn "FORK(austinhirsch)" . --include='*.rs' --include='*.ts' --include='*.tsx' --include='*.dart' | wc -l
#    compare against the count recorded in CUSTOMIZATIONS.md

# 6. PR into OUR main
git push -u origin HEAD
gh pr create --repo austinhirsch/buzz --base main \
  --title "Sync upstream block/buzz $(date +%F)" \
  --body "Merges upstream/main @ $(git rev-parse --short upstream/main). Conflicts: <list>. CI: <result>."
```

Merge sync PRs with a **merge commit**, not a squash. Squashing removes the
upstream commit history, and the next sync would then try to merge every
upstream commit again.

### Conflict resolution rules

1. **Upstream code wins by default.** Re-apply our change on top of their new
   version. Do not keep our old version of their code.
2. If upstream refactored the code our hook depended on, **do not restore the
   old structure.** Move the hook to the new equivalent spot, and update the
   ledger entry.
3. If upstream now does what our customization did, **delete our
   customization** and its ledger entry.
4. If a resolution is not obvious, stop and open a draft PR describing the
   conflict. Do not guess.

### Reading upstream before merging

For anything touching our customizations, read the upstream PRs, not just the
diff. Upstream's `VISION*.md` documents describe where the product is going.
If upstream is moving toward something a customization of ours works against,
catch that early.

---

## 6. Contributing back to upstream

To contribute a change (Tier 5):

```bash
git fetch upstream
git switch -c upstream-pr/<topic> upstream/main    # from upstream, never from our main
# ... make the change following AGENTS.md fully (Review-Proven Rules, tests, just ci)
git commit -s                                      # DCO sign-off required by block/buzz
git push -u origin HEAD
gh pr create --repo block/buzz --base main --head austinhirsch:upstream-pr/<topic>
```

- The branch must contain **no fork customizations**. Upstream reviewers would
  see them, and they would leak our changes.
- Read `CONTRIBUTING.md` and the applicable `VISION_*.md` documents first.
  AGENTS.md's Product Contract applies.
- After upstream merges the change, the next sync brings it in. Then remove our
  copy and its ledger entry.

---

## 7. Everyday customization workflow

1. **Tier it.** Choose the lowest tier from §3 that works.
2. **Sync first** if the last sync was more than a few days ago.
3. `git switch main && git pull && git switch -c fork/<topic>`
4. Build it. Follow **all** of AGENTS.md: Review-Proven Rules, rem-only text,
   `resetCommunityState`, no `StatefulWidget`, no `unwrap()`, and so on. Upstream's
   rules still apply to our code, both because they prevent real bugs and
   because following upstream's conventions makes later syncs easier.
5. Mark any Tier 3 edits and add a `CUSTOMIZATIONS.md` entry **in the same PR**.
6. `just ci`
7. `gh pr create --repo austinhirsch/buzz --base main`
8. The PR description must state the **tier** and list **any upstream files
   touched**.

### PR checklist

- [ ] Tier stated; no lower tier would work
- [ ] New code lives in a fork-owned namespace (§3 Tier 2)
- [ ] Every edited upstream file line is `FORK(austinhirsch)`-marked
- [ ] `fork/CUSTOMIZATIONS.md` updated (new row, or row removed)
- [ ] No incidental reformatting of upstream files (`git diff main --stat` reviewed)
- [ ] No new migrations (or §4.1 discussion linked)
- [ ] No edits to `AGENTS.md`, `.github/workflows/*`, lockfiles by hand
- [ ] `just ci` green
- [ ] PR targets `austinhirsch/buzz`, not `block/buzz`

---

## 8. Monitoring divergence

Run these periodically. Divergence that keeps growing costs more at every sync.

```bash
# Upstream files we've changed (excludes fork-owned paths)
git diff --stat upstream/main...main -- . ':!fork' ':!crates/fork-*' \
  ':!desktop/src/features/fork-*' ':!desktop/src-tauri/src/fork' ':!mobile/lib/features/fork_*'

# How far behind upstream we are
git rev-list --count main..upstream/main
```

**Keep upstream-file edits under about 20 files.** If the count grows past that,
find customizations that can move to a lower tier or be contributed upstream.
