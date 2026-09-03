# eng-metrics-suite

A self-hosted engineering metrics pipeline: import commit and PR/MR stats
from your GitHub, GitLab, Bitbucket, or Azure DevOps org into Postgres,
and generate a PDF report (team/author activity, PR review health, cycle
time, repo trends) from it.

This repo is just the `docker-compose.yml` needed to run it — the
application images (`git-processor`, `pr-processor`, `eng-reports`, and
optionally `issue-processor` for change-failure-rate/MTTR) are published
pre-built at [ghcr.io/gitultrahq](https://github.com/GitUltraHQ?tab=packages).
Free to use; see [LICENSE](LICENSE).

**Full docs: [gitultrahq.github.io/eng-metrics-docs](https://gitultrahq.github.io/eng-metrics-docs/)**

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

See [Getting Started](https://gitultrahq.github.io/eng-metrics-docs/getting-started/)
for why those two `mkdir`s matter, then
[Discovering Repos](https://gitultrahq.github.io/eng-metrics-docs/discovering-repos/)
to actually queue something up.

### Optional: Change Failure Rate / MTTR via Jira

`issue-processor` idles cleanly if unconfigured (same as
`git-processor`/`pr-processor` with an empty queue), so it's safe to leave
running even if you don't use this. To enable it: set the `JIRA_*`
variables in `.env`, then drop a `jira_project_map.yaml` (see
[issue-processor](https://github.com/GitUltraHQ/issue-processor)'s
`jira_project_map.example.yaml`) into `/var/lib/eng-metrics-suite/` and
run its seed script once:

```
docker compose run --rm issue-processor python3 discover_jira_projects.py \
    --config /var/lib/eng-metrics-suite/jira_project_map.yaml
```

This is a hand-maintained mapping, not auto-discovered — there's no Jira
API that can derive which project's Fix Versions apply to which repo.

## Requirements

- Docker + Docker Compose
