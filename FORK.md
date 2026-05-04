# Fork notes

This is a personal fork of [obsidian-git](https://github.com/denolehov/obsidian-git).

## Keeping the fork in sync with upstream

### One-time setup — add upstream remote

```bash
git remote add upstream https://github.com/denolehov/obsidian-git.git
git remote -v  # verify: origin = your fork, upstream = original repo
```

### Regular sync flow — when upstream releases a new version

**Step 1 — Fetch upstream changes**
```bash
git fetch upstream
```

**Step 2 — Update master to mirror upstream exactly**
```bash
git checkout master
git merge upstream/master
git push origin master
```

**Step 3 — Rebase your feature branch on top of updated master**
```bash
git checkout feature/vault-separation
git rebase master
```

Resolve any conflicts if they appear (unlikely since vault roots touches isolated files), then:
```bash
git push origin feature/vault-separation --force-with-lease
```

**Step 4 — Bump `manifest.json` version to match the tag**

BRAT compares the installed version (read from `manifest.json`) to the release tag. They must match exactly.

Edit `manifest.json`:
```json
"version": "2.39.0-vault-separation-0.0.1"
```

Commit the change:
```bash
git add manifest.json
git commit -m "chore: bump manifest to 2.39.0-vault-separation-0.0.1"
```

**Step 5 — Tag the new commit and push**

```bash
git tag 2.39.0-vault-separation-0.0.1
git push origin feature/vault-separation --tags
```

If you need to redo a tag (e.g. manifest was wrong), move it to the new commit:
```bash
git tag -d 2.39.0-vault-separation-0.0.1          # delete locally
git push origin :refs/tags/2.39.0-vault-separation-0.0.1  # delete remotely
git tag 2.39.0-vault-separation-0.0.1              # recreate on current commit
git push origin feature/vault-separation --tags
```

GitHub Actions builds and publishes the release automatically. BRAT picks it up for anyone using this fork.

---

### Summary

| Branch | Purpose |
|--------|---------|
| `master` | Clean mirror of upstream — never commit your changes here |
| `feature/vault-separation` | Your working branch — all releases tagged here |

### Why `--force-with-lease` and not `--force`

After a rebase the branch history is rewritten, so a force push is required. `--force-with-lease` is safer — it fails if someone else pushed to the branch in the meantime, preventing accidental overwrites.

---

## Installation via BRAT

The easiest way to install this fork without building from source is [BRAT](https://github.com/TfTHacker/obsidian42-brat) (Beta Reviewers Auto-update Tool).

1. Install **BRAT** from the Obsidian community plugins
2. Open BRAT settings → **Add beta plugin**
3. Paste this repo's GitHub URL (e.g. `https://github.com/<your-username>/obsidian-git`)
4. Click **Add plugin** — BRAT installs it and keeps it updated automatically

BRAT handles downloading `main.js` and `manifest.json` directly from GitHub releases, so no build step is needed on your end.

## Vault Roots

Adds the concept of **vault roots** — named folder scopes that control which parts of the vault are staged on auto commit-and-sync.

### Why

A vault often contains both personal notes and plugin source code (or other folders you don't want auto-committed). Vault roots let you declare exactly which folders participate in automatic git operations.

### What changed

| File | Change |
|------|--------|
| `src/types.ts` | Added `SyncRoot` interface `{ path, label }` and `syncRoots: SyncRoot[]` to settings |
| `src/constants.ts` | Default `syncRoots: []` (empty = stage everything, existing behavior) |
| `src/automaticsManager.ts` | On auto commit-and-sync, stages only configured roots before committing |
| `src/setting/settings.ts` | "Vault roots" section in Settings → Automatic with add/remove list UI |
| `src/commands.ts` | Added "Commit-and-sync vault roots" command (only visible when roots are configured) |

### How to configure

1. Build: `pnpm run build`
2. Copy `main.js` + `manifest.json` to `<vault>/.obsidian/plugins/obsidian-git/`
3. Reload plugin in Obsidian
4. Settings → Obsidian Git → **Vault roots** → click "Add vault root"
5. Enter a label (e.g. `Personal notes`) and path (e.g. `brain`)

When roots are configured, auto commit-and-sync stages only those folders. Manually staged files are always included regardless.

### Behavior when no roots are configured

Empty `syncRoots` = original behavior. All changed files are staged on auto commit-and-sync.
