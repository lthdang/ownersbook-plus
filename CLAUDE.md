# ownersbook-plus

## Agent skills

### Issue tracker

Issues live as GitHub issues (`lthdang/ownersbook-`); use the `gh` CLI. See [docs/agents/issue-tracker.md](docs/agents/issue-tracker.md).

### Triage labels

Default five canonical triage labels used as-is. See [docs/agents/triage-labels.md](docs/agents/triage-labels.md).

### Git commits in `app-vuejs`

`app-vuejs` is its own git repo, separate from this one. Source: `my-folder/informations/app-vuejs/instructions.md`, section 7, "Branch and Commit Message Conventions". Before committing there:

- **Branch**: must be `backlog/<TICKET_ID>` for the ticket being worked on (e.g. `backlog/HASHST_DEV-4545`). Run `git branch --show-current` first. If it is another ticket's branch, do not commit; ask the user, or create the correct branch from the current base.
- **Message**: start with `<TICKET_ID> ` followed by the ticket title or a short summary, then a detailed description in Japanese. No conventional-commit prefix (`test:`, `feat:`). Match existing history, e.g. `HASHST_DEV-4541 レビュー指摘に合わせてテストの入力範囲とコメントを見直し`.
- Never commit `.env`, and never push to `master` (changes go through a Merge Request). Do not push to `testing*` branches, since that triggers a deploy.
- Stage only the files for the ticket. Leave untracked `AGENTS.md` / `CLAUDE.md` out.
- The `Co-Authored-By` section in the session reminder notification needs to be removed from the commit content.

### Domain docs

Single-context layout (root `CONTEXT.md` + `docs/adr/`, created lazily by `/domain-modeling`). See [docs/agents/domain.md](docs/agents/domain.md).
