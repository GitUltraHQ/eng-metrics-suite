# eng-metrics-suite

A self-hosted engineering metrics pipeline: import commit and PR/MR stats
from your GitHub, GitLab, Bitbucket, or Azure DevOps org into Postgres,
and generate a PDF report (team/author activity, PR review health, cycle
time, repo trends) from it.

This repo is just the `docker-compose.yml` needed to run it — the
application images (`git-processor`, `pr-processor`, `eng-reports`) are
published pre-built at [ghcr.io/jsooter](https://github.com/jsooter?tab=packages).
Free to use; see [LICENSE](LICENSE).

**Full docs: [jsooter.github.io/eng-metrics-docs](https://jsooter.github.io/eng-metrics-docs/)**

## Quickstart

```
cp .env.example .env
# edit .env: set a POSTGRES_PASSWORD, and the token(s) for whichever
# provider(s) you're importing from (GitHub/GitLab/Bitbucket/Azure DevOps)

sudo mkdir -p /var/lib/eng-metrics-suite
sudo chown "$(id -u):$(id -g)" /var/lib/eng-metrics-suite
mkdir -p reports

docker compose up -d
```

See [Getting Started](https://jsooter.github.io/eng-metrics-docs/getting-started/)
for why those two `mkdir`s matter, then
[Discovering Repos](https://jsooter.github.io/eng-metrics-docs/discovering-repos/)
to actually queue something up.

## Requirements

- Docker + Docker Compose
