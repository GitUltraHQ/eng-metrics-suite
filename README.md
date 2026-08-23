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

docker compose up -d
```

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

### 2. Let the workers run

`git-processor` and `pr-processor` (already running from `docker compose
up`) pick up queued repos automatically and import commit/PR stats into
Postgres. Watch progress with:

```
docker compose logs -f git-processor pr-processor
```

### 3. Generate a report

```
docker compose run --rm eng-reports report.py --output /out/report.pdf
```

The PDF lands in `./reports/report.pdf` on the host (that directory is
bind-mounted into the container). Useful flags:

- `--period last-week|last-month|last-quarter` or `--start YYYY-MM-DD --end YYYY-MM-DD`
  (default: last 30 days)
- `--repo identity_key` (repeatable) and/or `--org github.com/owner` to
  scope the report instead of covering every repo you've imported
- `--team-map /out/teams.yaml` to roll up commit activity by team instead
  of by individual author. Put a YAML file mapping author email (or PR
  username) to team name at `./reports/teams.yaml` on the host, e.g.:
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
