# Privee deployment template

Deploys your own [Privee](https://github.com/MaxDac/Privee) server to
[Fly.io](https://fly.io) from the prebuilt image, without forking the code.
This repository holds only your configuration: [`fly.toml`](fly.toml) and the
[deploy workflow](.github/workflows/deploy.yml). It is yours to change.

**Full walkthrough:**
[Self-hosting Privee](https://github.com/MaxDac/Privee/blob/main/docs/self-hosting.md).

## Quick start

1. Click **Use this template** → **Create a new repository**.
2. Create the Fly app, a database and the secrets:

   ```bash
   fly apps create <app>
   fly mpg create                            # or any PostgreSQL; see the walkthrough
   fly mpg attach <cluster id> -a <app>      # sets DATABASE_URL
   fly secrets set SECRET_KEY_BASE=$(openssl rand -base64 48) -a <app>
   fly tokens create deploy -a <app>
   ```

3. In your repository, **Settings → Secrets and variables → Actions**:
   - secret `FLY_API_TOKEN`: the deploy token;
   - variable `FLY_APP`: the app name.
4. Run **Actions → Deploy → Run workflow**, then open
   `https://<app>.fly.dev/api/app/info`.

## Settings

Repository variables read by the deploy workflow:

| Variable | Default | Description |
| --- | --- | --- |
| `FLY_APP` | — (required) | Fly app name. Deploys are skipped until it is set. |
| `PHX_HOST` | `<FLY_APP>.fly.dev` | Public DNS name, e.g. your custom domain. |
| `PRIVEE_IMAGE` | `ghcr.io/maxdac/privee:latest` | Image to deploy. Pin a version (`ghcr.io/maxdac/privee:1.2.3`) or use `:main` for the latest commit. |
| `PRIVEE_INSTANCE_NAME` | — | Display name reported to clients. |
| `PRIVEE_SOURCE_URL` | the image's source repository | Source code of the version you run (AGPL-3.0). |

Change [`fly.toml`](fly.toml) for region, machine size and auto-stop. Every push
to `main` deploys.

## License

[AGPL-3.0-only](LICENSE), like Privee.
