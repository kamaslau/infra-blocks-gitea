# infra-blocks-gitea

[![Blocks](https://img.shields.io/badge/infra--blocks-index-555?style=flat-square)](https://github.com/kamaslau/infra-blocks)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](https://opensource.org/licenses/MIT)
![Repository size](https://img.shields.io/github/repo-size/kamaslau/infra-blocks-gitea?color=56BEB8)

A unit, not a demo: one service, one port, one documented credential, disposable with a single command.

Typical use: the code cannot leave the intranet, or a contractor handover needs a repository the client owns and can open in a browser.

[Gitea](https://docs.gitea.com/) — self-hosted Git with issues, pull requests and Actions, on SQLite until the team outgrows it.

## Service URL

- Web UI and Git over HTTPS `http://localhost:3000`
- Git over SSH `ssh://git@localhost:2222`

Ports can be changed in `.env`; see Usage. Port 3000 is the same default as Grafana in [infra-blocks-prometheus](https://github.com/kamaslau/infra-blocks-prometheus) — on one machine, change `PORT` for one of them rather than discovering the collision through a login page that answers with the wrong application.

## Default user

None. Gitea's registration form is switched off in this unit, so accounts are created from the command line — which is the point: a self-hosted Git that anyone who can reach the port may join is a Git server with no access control.

```bash
docker exec -u git gitea gitea admin user create \
  --admin --username admin --password 'change-this-now' \
  --email admin@example.com --must-change-password=false
```

Defaults are for local use only. `--must-change-password=false` matters in a script; without it the account is flagged and the first login is a password form.

## Usage

### Start with [Docker Compose](https://docs.docker.com/compose/)

```bash
# Initiate .env file
cp .env.example .env
# Start services
docker compose up -d
```

Update existing composed containers with latest images:

```bash
docker compose pull && \
docker compose down && \
docker compose up -d
```

### Configure the addresses

Two variables decide what the UI hands out, and both are wrong for a real host until you change them:

| Variable | Default | What it controls |
| -------- | ------- | ---------------- |
| `ROOT_URL` | `http://localhost:3000/` | Every web URL and HTTPS clone URL Gitea prints |
| `SSH_DOMAIN` | `localhost` | The host name in SSH clone URLs |
| `PORT_SSH` | `2222` | The published SSH port, and the port in clone URLs |

Setting `ROOT_URL` after the fact is not enough — the value is written into the config inside the volume at first start, so `docker compose down -v` is what makes a new value take effect on an existing instance.

### Migrate an existing repository

The web UI at `/repo/migrate` pulls from another Git host, including private ones with a token. For a one-off move, `git clone --mirror` followed by `git push --mirror` does the same thing without credentials in a form.

### CI runners

```bash
docker run --name gitea-runner -d --restart unless-stopped \
  -e GITEA_INSTANCE_URL=http://gitea:3000 \
  -e GITEA_RUNNER_REGISTRATION_TOKEN=<token-from-/admin/actions/runners> \
  -v /var/run/docker.sock:/var/run/docker.sock \
  gitea/act_runner
```

The registration token comes from `https://your-gitea.example.com/admin/actions/runners`. Mounting the Docker socket gives the runner root on the host — that is how act_runner works, and it is worth deciding deliberately rather than by copy-paste.

## Verify

```bash
curl -s http://localhost:3000/api/healthz
```

`"status": "pass"` with a `database:ping` entry inside means the process is up **and** the database answers. A `pass` with no database check is the install wizard, not an installed instance.

`docker compose ps` should say `(healthy)`, and `curl -s http://localhost:3000/user/sign_up | grep -o 'Registration is disabled'` proves the account policy is the one in `compose.yml` rather than the default.

The end-to-end check, after creating the admin account above:

```bash
curl -s -o /dev/null -w '%{http_code}\n' \
  -X POST -H 'Content-Type: application/json' \
  -d '{"name":"probe","private":true}' \
  -u admin:'change-this-now' \
  http://localhost:3000/api/v1/user/repos
```

`201` is a repository created by an authenticated user over the API, which exercises login, the database and the repository model in one request.

## Why this shape

Configuration is passed as `GITEA__section__key` environment variables, not as a tracked `app.ini`. Gitea rewrites that file on every start — it normalises spacing, persists what it was given, and generates `JWT_SECRET` on first boot. A tracked `app.ini` therefore ends up dirty in `git status` after the first `up`, and the generated secret inside it ends up committed. With environment variables the file lives in the `gitea_data` volume, the secret stays out of git, and the whole configuration is the `environment:` block of `compose.yml`, which is the file a reviewer actually reads.

`INSTALL_LOCK` is set, so a fresh volume boots straight into the application. Without it the first visitor gets the install wizard, and any settings passed to the container are ignored until someone clicks through it.

`START_SSH_SERVER` is deliberately **not** set. The standard image runs OpenSSH under s6 and Gitea only writes the `authorized_keys` for it; asking Gitea to run its own SSH server as well makes it bind port 22 inside the container, where OpenSSH already is, and the container restarts in a loop with `bind: address already in use`.

SQLite is the default database and there is no second container. The question this unit answers is "can the team keep their code here", and SQLite answers it up to a few dozen users; moving to PostgreSQL later is a `GITEA__database__*` block and a dump, not a reinstall.

`FORCE_PRIVATE=true` makes every new repository private, including the one created by a migration. Visible-by-default is the wrong default for a repository someone moved off a public host in a hurry.

## Notes for deployments in China

The `gitea/gitea` image pulls from Docker Hub and needs a registry mirror. `daemon.json` in this repository is a ready `/etc/docker/daemon.json` with four mirrors; copy it in and restart Docker, then confirm with `docker info | grep -A4 'Registry Mirrors'`.

Actions runners pull from `gitea.com` and Docker Hub, and the `act_runner` image is large: pull it before the machine moves behind the client's firewall, or the first workflow sits in `Waiting for a runner` with no error anywhere.

If the instance is served over HTTPS through [infra-blocks-caddy](https://github.com/kamaslau/infra-blocks-caddy), set `ROOT_URL` to the public URL and `SSH_DOMAIN` to the public hostname; the reverse proxy does not rewrite clone URLs, Gitea generates them from those two values.

---

Part of [infrastructure blocks](https://github.com/kamaslau/infra-blocks) · ← back to index
