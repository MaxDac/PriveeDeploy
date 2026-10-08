# PriveeDeploy

[![CI](https://github.com/MaxDac/PriveeDeploy/actions/workflows/ci.yml/badge.svg)](https://github.com/MaxDac/PriveeDeploy/actions/workflows/ci.yml)

Deploys [Privee](https://github.com/MaxDac/Privee) to [Fly.io](https://fly.io).

[Privee](https://github.com/MaxDac/Privee) holds the code and runs its checks;
it never deploys. This repository holds the Fly.io configuration
([`fly.toml`](fly.toml)) and a manual **Deploy** workflow that checks out Privee
at the commit you choose, builds its `Dockerfile` on Fly's remote builders and
deploys it.

- **Upstream** (`MaxDac/PriveeDeploy`) deploys the `bauta` instance, with the
  maintainer's Fly.io token stored as a repository secret.
- **Everyone else** forks this repository and adds their own Fly.io keys. Your
  fork deploys your own server; nothing in Privee or in the
  [Privee Android app](https://github.com/MaxDac/PriveeApp) has to change,
  because clients just ask for the server address.

Running Privee elsewhere (Docker, Kubernetes, other clouds) is up to you; see
[Self-hosting](https://github.com/MaxDac/Privee/blob/main/docs/self-hosting.md)
for the configuration reference and notes.

## Walkthrough

### 1. Prerequisites

- A [Fly.io account](https://fly.io/app/sign-up) with a payment method (needed
  even when usage stays within the free allowance).
- [`flyctl`](https://fly.io/docs/flyctl/install/), logged in with `fly auth login`.
- A GitHub account.

### 2. Fork this repository

Click **Fork** on GitHub. In your fork, open the **Actions** tab and enable
workflows (GitHub disables them in new forks).

### 3. Create the Fly.io app

Pick a globally unique name; it becomes `<name>.fly.dev`.

```bash
fly apps create <app>
```

To use another region than Amsterdam, change `primary_region` in your fork's
[`fly.toml`](fly.toml) ([regions](https://fly.io/docs/reference/regions/)).

### 4. Create the database

Use Fly's [Managed Postgres](https://fly.io/docs/mpg/) (or any Postgres you
can reach from Fly):

```bash
fly mpg create --name <app>-db --region ams
fly mpg attach <cluster id> -a <app>   # sets the DATABASE_URL secret
```

With an external database, set it yourself:
`fly secrets set DATABASE_URL=ecto://user:pass@host/db -a <app>`.

### 5. Set the secret key

```bash
fly secrets set SECRET_KEY_BASE=$(openssl rand -base64 48) -a <app>
```

### 6. Connect GitHub to Fly.io

Create a deploy token scoped to the app:

```bash
fly tokens create deploy -a <app>
```

In your fork, under **Settings → Secrets and variables → Actions**:

| Kind | Name | Value |
| --- | --- | --- |
| Secret | `FLY_API_TOKEN` | The token from the command above (the whole `FlyV1 ...` string). |
| Variable | `FLY_APP` | Your app name. |
| Variable | `PHX_HOST` | *Optional.* Public DNS name, defaults to `<app>.fly.dev`. |
| Variable | `PRIVEE_REPO` | *Optional.* Privee repository to deploy, defaults to `MaxDac/Privee`. |
| Variable | `PRIVEE_INSTANCE_NAME` | *Optional.* Display name reported to clients. |

With the GitHub CLI:

```bash
gh secret set FLY_API_TOKEN -R <you>/PriveeDeploy
gh variable set FLY_APP -b <app> -R <you>/PriveeDeploy
```

### 7. Deploy

Run **Actions → Deploy → Run workflow**, leaving `ref` as `main` (or use
`gh workflow run deploy.yml -R <you>/PriveeDeploy -f ref=main`). The first
build takes a few minutes; migrations run automatically before the new version
starts. The run summary shows the deployed commit and URL.

### 8. Check it works

```bash
curl https://<app>.fly.dev/api/app/info
```

It returns `"service": "privee"`. Open the URL in a browser to register, and
enter it as the server address in the Android app.

## Day-to-day

### Choosing what to deploy

The `ref` input accepts a branch, a tag or a commit SHA of Privee. Deploying a
full SHA that passed Privee's CI is the safest choice:

```bash
gh workflow run deploy.yml -R <you>/PriveeDeploy -f ref=<sha>
```

Deploys are never automatic: run the workflow whenever you want to update.
Roll back by deploying an older SHA, but check first that it doesn't need a
database migration that was rolled forward since.

### Custom domain

```bash
fly certs add chat.example.com -a <app>
```

Add the DNS records it prints, set the `PHX_HOST` variable to
`chat.example.com`, then deploy again.

### Running your own Privee fork

Fork Privee, set the `PRIVEE_REPO` variable to `<you>/Privee`, and deploy. The
instance links to the exact source it runs (`PRIVEE_SOURCE_URL`), which the
AGPL requires when you serve modified code to others; keep your fork public.

### Keeping your fork up to date

Use **Sync fork** on GitHub to pick up changes to this repository (such as
`fly.toml` tweaks); merge them into your own `fly.toml` edits if you have any.

### Cost and scale to zero

[`fly.toml`](fly.toml) lets Fly stop the machine when idle and restart it on
the next request, which keeps a small instance cheap. Privee keeps undelivered
encrypted messages in memory, so they are lost whenever the machine stops; for
an always-on server set `auto_stop_machines = 'off'` and
`min_machines_running = 1`. Always run a **single machine**: the workflow
deploys with `--ha=false`, don't scale it out.

## Deploying by hand

From a Privee checkout, with this repository's `fly.toml` copied into it:

```bash
fly deploy --remote-only --app <app> --ha=false --env PHX_HOST=<app>.fly.dev
```

## Checks

[`ci.yml`](.github/workflows/ci.yml) lints the workflows and checks that
`fly.toml` stays instance-agnostic (no `app` or `PHX_HOST`) and consistent.
Changes to `main` go through pull requests that must pass these checks.

## License

[GNU AGPL v3](LICENSE), like Privee.