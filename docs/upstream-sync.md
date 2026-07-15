# Upstream Sync Guide

This fork tracks [uxjulia/CrossInk](https://github.com/uxjulia/CrossInk) via the
`upstream-main` branch. The tracking branch was set up when merging v1.4.0 and
persists on `origin` so it's available for future updates.

## To merge a future upstream release

```bash
# 1. Fetch latest upstream changes and tags
git fetch upstream --tags

# 2. Update the tracking branch
git checkout upstream-main
git merge upstream/main

# 3. Push tracking branch to origin (preserve for next time)
git push origin upstream-main

# 4. Merge into your fork
git checkout main
git merge upstream-main
```

**Conflict resolution** — the fork's customizations are in:

| Area | Files | Strategy |
|------|-------|----------|
| Web file manager | `web/pages/files.html`, `web/pages/files.js` | Keep fork's version (`git checkout --ours <file>`) |
| CI workflow | `.github/workflows/build-firmware-fork.yml` | Keep fork's version (`git checkout --ours <file>`) |
| README | `README.md` | Auto-merged; keep upstream unless fork section conflicts |
| Everything else | Upstream firmware code | Accept upstream (`git checkout --theirs <file>`) |

When a conflicted fork file needs to stay fork's version:
```bash
git checkout --ours web/pages/files.html
git checkout --ours web/pages/files.js
git add web/pages/files.html web/pages/files.js
```

```bash
# 5. Complete the merge
git merge --continue

# 6. Regenerate web headers (if web files changed)
python3 scripts/build_web.py

# 7. Push, tag, and let CI build
git push origin main --tags
```

The CI workflow (`.github/workflows/build-firmware-fork.yml`) will automatically
build on push to `main` and upload firmware artifacts.

## Initial setup (already done)

If cloning this fork fresh on a new machine:

```bash
git remote add upstream https://github.com/uxjulia/CrossInk
git fetch upstream --tags
git branch upstream-main upstream/main
git push origin upstream-main
```

## Remotes reference

| Remote | URL |
|--------|-----|
| `origin` | `https://github.com/binhex/CrossInk` |
| `upstream` | `https://github.com/uxjulia/CrossInk` |
