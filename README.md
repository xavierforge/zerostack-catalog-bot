# zerostack-catalog-bot

Weekly automation that keeps [zerostack](https://github.com/gi-dellav/zerostack)'s
embedded model catalog (`data/models.json`) in sync with
[models.dev](https://models.dev), by opening a pull request against upstream.

## Why a separate repo

Scheduled workflows only run on a repository's default branch. The
[xavierforge/zerostack](https://github.com/xavierforge/zerostack) fork keeps
`main` level with upstream (fast-forward only), so it cannot carry a fork-only
workflow commit. The automation lives here instead, and the fork stays a clean
mirror.

## How it works

Every Monday 06:00 UTC (or on manual dispatch), `update-models.yml`:

1. Checks out upstream `gi-dellav/zerostack` at `main`.
2. Runs upstream's own `scripts/gen-models-catalog.sh` to regenerate
   `data/models.json`.
3. Stops if nothing changed.
4. Renders a fixed-format change summary (`scripts/catalog-diff.sh`): a
   per-provider totals table, then Added/Removed/Changed lists with model
   names, context sizes, and pricing.
5. Force-pushes the result to the fork's `chore/refresh-model-catalog` branch
   (always based on current upstream `main`).
6. Opens a PR on upstream from that branch, or refreshes the body of the open
   one. The PR always represents the full pending diff against upstream.

## Setup

Two secrets are required.

`FORK_SYNC_PAT` is a fine-grained personal access token that syncs the fork's
default branch and pushes the refresh branch. Create it under GitHub Settings,
Developer settings, Personal access tokens, Fine-grained tokens, with:

- Resource owner: `xavierforge`
- Expiration: no expiration
- Repository access: only `xavierforge/zerostack`
- Repository permissions: Contents (read and write), Workflows (read and write)

Workflows write is needed because upstream commits routinely touch
`.github/workflows/` and GitHub rejects any push or fork sync carrying such a
commit from a token without it.

`UPSTREAM_PR_PAT` is a classic personal access token with the `public_repo`
scope. It opens the PR on upstream as the token's owner. It has to stay
classic: fine-grained tokens cannot contribute to public repositories the
owner is not a member of. Until `FORK_SYNC_PAT` is set, the workflow falls
back to this token for the sync and push steps as well.

```sh
gh secret set FORK_SYNC_PAT --repo xavierforge/zerostack-catalog-bot
gh secret set UPSTREAM_PR_PAT --repo xavierforge/zerostack-catalog-bot
```

Trigger a run manually:

```sh
gh workflow run update-models.yml --repo xavierforge/zerostack-catalog-bot
```
