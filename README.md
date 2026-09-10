# production-deployments

This repository is the answer to one question:

> **What should the VPS be running right now?**

It contains no source code and no secrets. It is deliberately **PUBLIC**.

## Why public

The VPS reads these files over plain `https://raw.githubusercontent.com` with no
token at all. That is what keeps GitHub credentials off the server: the VPS has
no repository access, no SSH deploy key, and no `delete:packages` rights. It can
only read this file and pull the image it names.

**Never commit a secret here.** Connection strings, tokens and keys live in
`/opt/apps/prod/backend.env` on the VPS and nowhere else.

## Who writes it

Nobody by hand, normally. The `Build Landing` and `Build Backend` workflows in
the app repositories commit here using `DEPLOYMENTS_TOKEN` (a PAT scoped to
`contents: write` on this repository only). The commit log is therefore a
complete deployment history.

## Who reads it

- The VPS, every minute, via `/opt/apps/deploy/update-service.sh`.
- The `Prune GHCR` workflows, nightly, to learn which image tags must survive.

## Layout

```
landing/deployment.json          asanfsp.ir
backend-prod/deployment.json     api.asanfsp.ir
backend-dev/deployment.json      api-dev.asanfsp.ir
dashboard-prod/deployment.json   dashboard.asanfsp.ir      not published yet
dashboard-dev/deployment.json    dashboard-dev.asanfsp.ir  not published yet
```

## Fields

| Field          | Meaning                                                        |
| -------------- | -------------------------------------------------------------- |
| `image`        | Full image reference the VPS pulls.                              |
| `tag`          | Desired version. 12-char commit SHA. **This is the trigger.**    |
| `previous_tag` | Rollback target. Protected from the nightly GHCR prune.          |
| `commit`       | Full source SHA, so "what code is running?" always has an answer.|
| `migration`    | `none`, or `required` when the image changes the database schema.|
| `created`      | UTC build time.                                                  |

### `migration: required`

Set by the backend workflow when the commit message contains `[migration]`. The
VPS **refuses to deploy that tag** and keeps running the old one until the SQL
scripts have been applied by hand and the hold is cleared:

```bash
sudo touch /opt/apps/deploy/migrated/backend-prod-<tag>
```

The API does not migrate on boot. Without this gate an auto-deploy would run new
code against the old schema.

## Seed values

`0000000000aa` is a placeholder, not a real image. The matching `.env` on the VPS
is seeded with the same value so the first timer tick is a no-op rather than a
failing pull. The first real `[deploy]` commit replaces it.
