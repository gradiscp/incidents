# 2026-10-06 - CI red on the first push to main: paths-filter could not fetch the previous commit

**Status:** Resolved
**Area:** GitHub Actions CI, dorny/paths-filter, actions/checkout

## Summary

The first merge into `main` of a new company project repo turned CI red. The
`changes` job (dorny/paths-filter) has to compare the push with the previous
`main` commit using git. The depth-1 checkout did not contain that commit, and
fetching it failed because `persist-credentials: false` had removed the token
from the checkout. Fixed by checking out the full history in that job.

## Impact

- One red CI run on `main`. `changes` failed, so the app, API and image jobs
  were skipped and the summary job `ci` failed.
- Nothing deployed or lost: the repo has no deploy pipeline yet.
- A second PR (from the same, already merged branch) was needed for the fix.

## Environment

| Component | Detail |
|---|---|
| Repo | private company project, `.github/workflows/ci.yml`, triggers `pull_request` and `push` to `main` |
| Checkout | `actions/checkout` v7.0.1, default `fetch-depth` (1), `persist-credentials: false` |
| Filter | `dorny/paths-filter` v4.0.3 |
| History | `main` held only the initial commit before the merge |

## Timeline

Local time (CEST). Commit times from `git log`; the rest is estimated.

- 2026-10-06, morning - first feature PR → `main` merged (time not
  recorded). Its PR runs had not shown the problem (see *Root cause*).
- shortly after - the push run on `main` fails in `changes`.
- 10:00 *git* - fix committed on the same branch.
- later - second PR merged, `main` green (reported by the owner).

## Root cause

On `pull_request` events paths-filter takes the list of changed files from
the GitHub API. On `push` events it uses git and needs the previous commit
(`github.event.before`) locally. With the default depth-1 checkout only the new
commit was there, so paths-filter ran `git fetch` itself. The repo is private
and `persist-credentials: false` removes the token from the git config after
checkout, so that fetch had no credentials.

The push trigger only fires for `main`, and the merge was the first push to
`main`, so no earlier run exercised this path. Before the merge, the workflow
had only been checked with actionlint and a local replay of one job, and a
push to `main` cannot be replayed locally.

## Evidence

Log of the failing step (commit hash shortened to `<sha>`):

```
Changes will be detected between <sha> and main
Ensuring <sha> is fetched from origin
  /usr/bin/git cat-file -e <sha>^{commit}
  fatal: Not a valid object name <sha>^{commit}
  /usr/bin/git fetch --depth=1 --no-tags origin <sha>
  fatal: could not read Username for 'https://github.com': No such device or address
Error: The process 'git fetch --depth=1 --no-tags origin <sha>' failed with exit code 128
```

- `git log -1 <sha>` → `initial commit`, the tip of `main` before the merge.
- paths-filter README, push events: "Uses git commands to detect changes -
  repository must be already checked out".

## What didn't work

Nothing was tried before the fix. Rejected alternatives:

- Dropping `persist-credentials: false` in that job: would work, but leaves
  the token in `.git/config` for the rest of the job.
- `fetch-depth: 2`: enough when every push to `main` adds exactly one commit
  (merge commit or squash), but fails again for "Rebase and merge" of a
  multi-commit PR or a direct push of several commits.

## Resolution

Commit `ci: full history for paths-filter on push to main`: the checkout in
the `changes` job gets `fetch-depth: 0`, with a comment explaining why.
`persist-credentials: false` stays. The repo is a few MiB, so the full
history costs seconds.

## Unverified

- That the PR runs went through because paths-filter uses the GitHub API for
  `pull_request` events (paths-filter README; the PR run logs were not read).
- Which merge button was used for the merges.

## Prevention

- The fix and its reason live next to the setting: the comment above
  `fetch-depth: 0` in the `changes` job of `.github/workflows/ci.yml`.
- Rule of thumb: a workflow with a `push` trigger is only tested by a real
  push. After changing `ci.yml`, watch the first run on `main`, not only the
  PR run.
