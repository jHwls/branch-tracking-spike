# branch-tracking-spike

Miniature of the squid-fortress branch-tracking automation. Validates the GitHub Action that keeps a
long-lived upgrade branch in sync with the primary branch by replaying each merged PR's patch.

- Primary branch: `main`
- Long-lived upgrade branch: `spike/upgrade` (diverges from `main` only in `VERSION`)
- Workflow: `.github/workflows/sync-upgrade-branch.yml` (fires on a PR **merged into `main`**)

| File          | Role                                                                 |
| ------------- | ------------------------------------------------------------------- |
| `VERSION`     | Conflict surface — `spike/upgrade` already changed it (otp/elixir).  |
| `lib/app.txt` | Clean surface — feature PRs that touch only this sync without conflict. |

## One-time GitHub setup after pushing

1. Push both branches:
   ```
   git remote add origin git@github.com:<you>/branch-tracking-spike.git
   git push -u origin main
   git push origin spike/upgrade
   ```
2. Repo **Settings → Actions → General → Workflow permissions**:
   - Select **Read and write permissions**.
   - Check **Allow GitHub Actions to create and approve pull requests** (needed for the conflict PR).

## Test it

### Clean path (expect an automatic sync)

1. Branch off `main`, edit `lib/app.txt` (e.g. add a flag), open a PR into `main`, merge it.
2. The action cherry-picks the PR onto `sync/pr-<N>`, `--no-ff` merges it into `spike/upgrade`, and
   pushes. Confirm `spike/upgrade` now has a "Merge #<N> into spike/upgrade" commit containing the
   change, authored by whoever's commit landed on `main`.

### Conflict path (expect an adaptation PR)

1. Branch off `main`, change the `otp=` line in `VERSION`, open a PR into `main`, merge it.
2. The action commits the patch **with conflict markers** on `sync/pr-<N>` and opens an
   **"Adapt #<N> for spike/upgrade"** PR. Resolve the markers there; that resolution commit is the
   isolated upgrade-vs-main delta, and merging the PR is the second merge commit.

### What "good" looks like

```
git log --graph --oneline --all
```

Each synced PR shows two merge commits — one on `main`, one on `spike/upgrade` — and conflicts are
captured as a separate, reviewable resolution rather than entangled in the merge.
