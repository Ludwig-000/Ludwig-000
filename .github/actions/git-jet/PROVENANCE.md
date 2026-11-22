# Vendored Git Jet

This folder is a local copy of the Git Jet GitHub Action, so the profile
workflow no longer depends on any outside repository for it.

- **Original:** `PingPongPlus/git-jet` (deleted from GitHub in November 2026)
- **Commit:** `8e4643e55fbc05da7a60524b240ee77470061f2b` (2026-11-03, the
  commit the workflow was already pinned to)
- **Recovered from:** `niklasxi/git-jet`, which contains that exact commit.
  Git commit IDs are content hashes, so the code is identical to the original.
- **License:** MIT (see `LICENSE`)

## What was copied

Only the files the action executes:

| Path | Status |
| --- | --- |
| `src/git_jet/*.py` | byte-identical to the commit |
| `scripts/run_action.py` | byte-identical |
| `scripts/publish_assets.py` | modified (see below) |
| `LICENSE` | byte-identical |
| `action.yml` | modified (see below) |
| `requirements.txt` | new |

Tests, artwork drafts, demo GIFs and docs were left out; none run in CI.

## Changes to `action.yml`

1. `actions/setup-python@v7` is pinned to the commit for v7.1.0
   (`6d4fd22b20465ccc0f7354476588e8df7c9ca0e0`), so a moved tag can't change it.
2. Instead of `pip install`-ing the package (which built it with an unpinned
   `setuptools` and an unpinned `Pillow>=13,<14` from PyPI), it installs only
   `pillow==13.0.0` from a prebuilt wheel, checked against SHA-256 hashes, and
   runs the code straight from `src/` via `PYTHONPATH`.

## Change to `scripts/publish_assets.py`

The original added a new commit to the `output` branch on every run, so each
day's ~20 MB GIF stayed in git history forever (143 MB after 18 runs, on track
for several GB a year). Now every publish replaces `output` with a single
parentless commit holding just the current `git-jet.gif` and `git-jet.png`.

- The push is `--force-with-lease` pinned to the commit seen at the start, so
  if anything else updated `output` in the meantime the push is refused.
- Only `refs/heads/output` is ever pushed; `main` is never touched.
- If the images are unchanged and `output` is already a single commit, nothing
  is pushed. The comparison fetches commit/tree objects only, not the images.

## Security audit (2026-11-22)

Every Python file in the action was read and checked with a syntax-tree scan
of every function call. Findings:

- **Network:** only `https://api.github.com` (read-only `GET` of your public
  profile, repo list and commit list), with redirects refused, a 1000-request
  cap and a 10 MB response cap. No other hosts are contacted.
- **Token:** `GITHUB_TOKEN` is only sent as an `Authorization` header to
  `api.github.com`, and to `https://github.com/` by `git` in the publish step.
  It is never printed, logged or written to disk or git config. Git error
  output is suppressed in case it contains auth details.
- **Writes to your repo:** only `publish_assets.py`, and only the files
  `git-jet.gif` and `git-jet.png` to the `output` branch. It refuses to run if
  `output` is the default branch, never touches your checked-out files, and
  replaces `output` only with a lease-protected force-push (see above).
- **Local files:** writes only inside `git-jet-output/` in the workspace (path
  traversal is checked) and in temporary directories.
- **Processes:** the only `subprocess` call is `git`, in the publish step.
- **No** `eval`, `exec`, `pickle`, dynamic imports, `getattr` tricks, encoded
  blobs, hidden/bidirectional Unicode characters, or downloads of code.
- Everything else (`art.py`, `lincoln.py`, `water.py`, `flight.py`,
  `render.py`) only calls Pillow drawing functions and `math`.

## Updating

There is no upstream anymore. If you change anything here, review the diff
yourself, and re-pin hashes in `requirements.txt` when bumping Pillow
(`pip download pillow==X --no-deps` then `pip hash`, or copy the SHA-256s
from the PyPI release page).
