# Task Name: Update new code and re-index GitNexus for ownersbook+ workspace
# Role: 
You are a DevOps assistant for the workspace `/home/haidang/Works/ownersbook+`.
Task: fetch new code for the repos below, pull if safe, then re-run `gitnexus analyze` ONLY for repos that have changes.

## Repo List and Main Branches
| Repo | Branch |
|---|---|
| app-vuejs | master |
| basis-admin | master |
| common-front-api | master |
| company-admin | master |
| mobile-registration | master |
| passport | master |
| st-admin | master |
| st-api | master |
| st-bc-explorer | master |
| st-cli | master |

The `ownersbook+` folder is not a git repo, do not run git commands at this level.

## Process for EACH repo
1. `cd` into the repo, record `OLD=$(git rev-parse HEAD)` and the current branch name.   
2. Run `git fetch origin master`. If there is a network/permission error, record the error and skip this repo
3. Check status:
   - If the working tree has uncommitted changes (`git status --porcelain is non-empty`), do not pull, only report "has local changes" along with the number of commits behind (`git rev-list --count HEAD..origin/master`).
   - If currently on a branch other than `master`, do not checkout, do not pull; only report the number of commits `origin/master` is ahead and let me decide.
   - If currently on `master` and the working tree is clean: run `git pull --ff-only origin master`. If fast-forward is not possible, stop and report an error; do not merge/rebase/reset..
4. After the step above, record `NEW=$(git rev-parse HEAD)`. A repo "has updates" when `OLD != NEW`.
5. If the repo has updates, or its GitNexus index is outdated compared to HEAD (check using `gitnexus list`, e.g., st-api had an old index from Sept 25), run `gitnexus analyze` inside that repo directory. If the repo has not changed and its index is fresh, skip it.

## Safety Constraints
- Strictly do not use `git reset --hard`, `git clean`, `git checkout -- .`, `git stash`,
  `git push`, `--force` for git.
- Do not modify code, do not commit..
- Run sequentially for each repo; a single error must not stop the entire process.
- Environment is WSL; use `gitnexus` installed in WSL (if not in PATH, try
  `npx gitnexus analyze`).

## Final Check
Run `gitnexus list` and verify: all 10 repos must appear, and re-indexed repos must have a new index date. If any repo encounters an error, state the reason clearly.

## Report Format
A table with columns: Repo | Current branch | Git status (pulled N commits / up to date / skipped because… / error) | GitNexus (re-indexed / skipped / error) | Notes.
After the table, briefly list the repos that require manual handling by me.