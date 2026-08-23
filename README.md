# eng-metrics-suite

A self-hosted engineering metrics pipeline: import commit and PR/MR stats
from your GitHub, GitLab, or Bitbucket org into Postgres, and generate a
PDF report (team/author activity, PR review health, cycle time, repo
trends) from it.

This repo is just the `docker-compose.yml` needed to run it — the
application images (`git-processor`, `pr-processor`, `eng-reports`) are
published pre-built at [ghcr.io/jsooter](https://github.com/jsooter?tab=packages).
Free to use; see [LICENSE](LICENSE).

## Requirements

- Docker + Docker Compose

## Quickstart

```
cp .env.example .env
# edit .env: set a POSTGRES_PASSWORD, and the token(s) for whichever
# provider(s) you're importing from (GitHub/GitLab/Bitbucket)

sudo mkdir -p /var/lib/eng-metrics-suite
sudo chown "$(id -u):$(id -g)" /var/lib/eng-metrics-suite
mkdir -p reports

docker compose up -d
```

`/var/lib/eng-metrics-suite` is where config files (not secrets — those
go in `.env`) live: `discover_repos.py`'s `--config` YAML and
`report.py`'s `--team-map` YAML both get read from there, mounted
read-only into the relevant containers at the same path. It needs to
exist before `docker compose up` (Docker will auto-create it as
root-owned otherwise, which then blocks you from writing to it without
`sudo`).

`./reports` needs the same up-front `mkdir` for a different reason:
`eng-reports` runs as a non-root user (UID 1000) so its output isn't
root-owned on the host, but that only helps once the directory already
exists with the right owner — if Docker has to auto-create `./reports`
itself (e.g. because you skipped this step), it does so as root, and the
non-root container then can't write into it at all
(`PermissionError: [Errno 13] Permission denied`). If your host user
isn't UID 1000, add `--user "$(id -u):$(id -g)"` to the `docker compose
run` command in step 3 below instead.

This starts Postgres plus the `git-processor` and `pr-processor` workers.
The workers exit as soon as they find nothing queued to do — that's
expected, not a crash; `restart: unless-stopped` just means they check
again periodically rather than sitting in a busy loop. Nothing happens
until you queue some repos:

### 1. Discover repos

`git-processor` and `pr-processor`'s default command is their worker
(`worker.py`), so one-shot scripts like `discover_repos.py` need
`--entrypoint python3` to override that:

```
docker compose run --rm --entrypoint python3 git-processor discover_repos.py <org-or-group> --provider github
```

`--provider` is `github` (default), `gitlab`, `bitbucket_cloud`, or
`bitbucket_server`. `<org-or-group>` is an org/group/workspace, not a
personal account — this discovers everything the org owns, not one
user's repos. Queues each repo found — safe to re-run later to pick up
new ones.

To skip archived/inactive repos or ones matching a name pattern (e.g. bot
repos), write a `discover_config.yaml` to `/var/lib/eng-metrics-suite/`
on the host:

```yaml
exclude_archived: true            # skip repos the provider reports as archived
exclude_inactive_days: 730        # skip repos with no push in this many days
exclude_patterns:                 # skip repos whose "org/repo" name matches (regex, re.search)
  - "-bot$"
  - "^terraform-"
```

Then pass it via `--config`:

```
docker compose run --rm --entrypoint python3 git-processor discover_repos.py <org-or-group> --provider github --config /var/lib/eng-metrics-suite/discover_config.yaml
```

Coverage varies by provider: GitHub supports all three keys. Bitbucket
Cloud has no "archived" concept (`exclude_archived` never matches there)
and checks `updated_on` instead of push date for inactivity. Bitbucket
Server's repo-list endpoint exposes neither signal, so only
`exclude_patterns` does anything there.

### 2. Let the workers run

`git-processor` and `pr-processor` (already running from `docker compose
up`) pick up queued repos automatically and import commit/PR stats into
Postgres. Watch progress with:

```
docker compose logs -f git-processor pr-processor
```

### Scaling workers

There's no worker-count setting — `git-processor` and `pr-processor` each
claim one repo at a time from the shared queue (`FOR UPDATE SKIP LOCKED`,
so concurrent claims never collide), and by default `docker compose up`
runs exactly one of each. To process more repos in parallel, run more
instances with Compose's `--scale`:

```
docker compose up -d --scale git-processor=4 --scale pr-processor=4
```

Each replica still exits once the queue's empty and gets restarted
independently by `restart: unless-stopped`, so the scale factor persists
without further action. `--max-attempts`/`--lease-minutes` are per-repo,
not per-worker, so scaling up doesn't change retry behavior — it just
means more repos get claimed per pass.

### 3. Generate a report

```
docker compose run --rm eng-reports report.py --output /out/report.pdf
```

If your host user isn't UID 1000 (check with `id -u`), add `--user
"$(id -u):$(id -g)"` so the output file comes out owned by you instead of
UID 1000:

```
docker compose run --rm --user "$(id -u):$(id -g)" eng-reports report.py --output /out/report.pdf
```

The PDF lands in `./reports/report.pdf` on the host (that directory is
bind-mounted into the container). Useful flags:

- `--period last-week|last-month|last-quarter` or `--start YYYY-MM-DD --end YYYY-MM-DD`
  (default: last 30 days)
- `--repo identity_key` (repeatable) and/or `--org github.com/owner` to
  scope the report instead of covering every repo you've imported
- `--team-map /var/lib/eng-metrics-suite/teams.yaml` to roll up commit
  activity by team instead of by individual author. Put a YAML file
  mapping author email (or PR username) to team name at
  `/var/lib/eng-metrics-suite/teams.yaml` on the host, e.g.:
  ```yaml
  alice@example.com: Platform
  bob@example.com: Platform
  carol@example.com: Product
  ```

### Scheduling reports

`docker compose run` is a normal one-shot command — wire it into a host
cron job or systemd timer to get a report on a recurring schedule (e.g.
weekly, for a Monday-morning exec digest):

```
0 8 * * 1 cd /path/to/eng-metrics-suite && docker compose run --rm eng-reports report.py --period last-week --output /out/weekly-report.pdf
```

## Updating

Images are rebuilt and republished automatically. Pull the latest and
recreate:

```
docker compose pull
docker compose up -d
```
