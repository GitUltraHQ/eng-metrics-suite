---
name: gitultra-setup
description: Automates end-to-end installation of GitUltra (eng-metrics-suite or eng-metrics-suite-pro) -- environment detection, .env setup, docker compose up, repo discovery, optional Jira integration, and a verified sample report. Use when a user asks to set up, install, or configure GitUltra.
---

# GitUltra Setup

This skill is an executable version of the manual install docs at
https://gitultrahq.github.io/eng-metrics-docs/ (Getting Started →
Discovering Repos → Jira Integration → Running Workers → Generating
Reports). If anything here seems to disagree with those docs, the docs
are the source of truth -- flag the discrepancy rather than silently
picking one.

**Core rule: never just run commands and declare success.** Every step
below has a real verification action attached. The final step is not
optional -- a customer should walk away with a real generated report in
hand, not just a "setup complete" message.

## Step 1: Detect the environment

Run these checks before asking the user anything:

1. **Which tier is this?** Check whether `docker-compose.yml` in the
   current directory references an `eng-api` service (`grep -q
   "eng-api:" docker-compose.yml`). If yes, this is
   `eng-metrics-suite-pro` (paid tier) -- follow the "Pro tier only"
   notes below in addition to everything else. If no `docker-compose.yml`
   exists at all, or it doesn't look like this repo, stop and ask the
   user to `cd` into their `eng-metrics-suite`/`eng-metrics-suite-pro`
   checkout first (or clone it) -- don't guess a location.
2. **Prerequisites**: check `docker --version` and `docker compose
   version`. If either is missing, stop and tell the user what to
   install -- don't attempt to install Docker itself, that's out of
   scope and too environment-specific to automate safely.
3. **Fresh install or upgrade?** Check whether `.env` already exists.
   - If it doesn't: this is a fresh install, proceed to Step 2.
   - If it does: this is likely an upgrade or a re-run. Read the
     existing `.env` (don't overwrite it blindly) and check `docker
     compose ps` -- if containers are already running, this is
     probably "pull the latest images and restart," not a from-scratch
     setup. Confirm with the user which they want before proceeding
     (`git pull && docker compose pull && docker compose up -d`
     covers the upgrade case; skip straight to Step 6 verification
     afterward).

## Step 2: Ask only what can't be inferred

Everything else gets a sensible default or is derived -- don't prompt
for it. Ask (in one pass, not one question at a time):

- **Org/group name and provider** (GitHub org, GitLab group, Bitbucket
  Cloud workspace, Bitbucket Server project, or Azure DevOps
  organization) -- this can't be inferred, it's specific to the
  customer.
- **Provider token(s)** for whichever provider they named (`GITHUB_TOKEN`,
  `GITLAB_TOKEN`, `BITBUCKET_USERNAME`+`BITBUCKET_APP_PASSWORD`,
  `BITBUCKET_SERVER_USERNAME`+`BITBUCKET_SERVER_TOKEN`, or
  `AZURE_DEVOPS_PAT`). These go straight into the user's own local
  `.env` file on their own machine -- never echo them back in chat
  after they're entered, and never put them anywhere except that file.
- **Which optional integrations to enable**: ask plainly, as a single
  multi-select-style question -- "Change failure rate/MTTR (needs
  Jira)?", "Investment allocation + planning quality signals report
  (needs Jira)?", "Team rollups (needs a roster CSV)?". Each is
  independently optional; don't assume yes or no.
  - If either Jira option is yes: ask for `JIRA_BASE_URL`, `JIRA_EMAIL`,
    `JIRA_API_TOKEN` once (shared by both flows), then the specific
    vars each flow needs (see Step 4).
  - If team rollups: ask where their roster data lives (spreadsheet,
    HR system, etc.) and help them export it to the required
    `email,team` CSV shape -- don't assume they already have the file
    in the right format.
- **Repo filtering**: ask if they want to exclude archived/inactive
  repos or name patterns (see Step 4's `discover_config.yaml`) -- default
  to no filtering if they don't care, don't force this decision.

**Pro tier only**: also ask for `GITULTRA_LICENSE_KEY` (issued by
GitUltra after purchase -- if they don't have one, stop and point them
to support@gitultra.com rather than guessing or proceeding without it).
**Do not ask for `ENG_API_KEY`** -- generate it yourself with `openssl
rand -hex 32` and write it into `.env` directly; this is the customer's
own credential for their own future dashboard integrations, not
something they need to have an opinion about at install time. Tell them
afterward where to find it in `.env` if they want to use it later.

## Step 3: Write configuration

1. `cp .env.example .env` if it doesn't already exist.
2. Fill in every value gathered in Step 2 directly into `.env` (edit
   the file, don't just tell the user to do it by hand -- that's the
   whole point of this skill). Leave anything not configured blank,
   matching `.env.example`'s own convention of "blank means disabled,"
   not a placeholder value.
3. Create the two required host directories *before* `docker compose
   up` -- if Docker auto-creates them itself it does so as root, which
   then blocks non-root writes:
   ```
   sudo mkdir -p /var/lib/eng-metrics-suite
   sudo chown "$(id -u):$(id -g)" /var/lib/eng-metrics-suite
   mkdir -p reports
   ```
   **Don't stop at "does the directory exist" -- check it's actually
   writable by the current user too** (`test -w reports && test -w
   /var/lib/eng-metrics-suite`), even if `mkdir -p` reported success.
   `mkdir -p` is a silent no-op on a path that already exists,
   regardless of who owns it -- a stale directory left root-owned from
   an earlier attempt (or from Docker auto-creating it before this
   skill ever got a chance to) will pass the `mkdir -p` step cleanly
   and then fail much later, confusingly, inside the container at
   report-generation time. If either directory isn't writable, `sudo
   chown "$(id -u):$(id -g)" <dir>` fixes it -- don't try to just
   recreate it and hope; confirm the actual owner is wrong first with
   `ls -la`.
   
   If `sudo` itself isn't available or requires an interactive password
   this session can't supply (a real, common environment constraint,
   not a bug in this skill), stop and ask the user to run the `sudo`
   commands themselves in their own terminal, rather than trying to
   work around it.
4. If team rollups were requested, write the `email,team` CSV to
   `/var/lib/eng-metrics-suite/teams.csv`.
5. If repo filtering was requested, write `discover_config.yaml` to
   `/var/lib/eng-metrics-suite/discover_config.yaml` (keys:
   `exclude_archived`, `exclude_inactive_days`, `exclude_patterns` --
   see the docs' "Discovering Repos" page for which keys each provider
   actually supports; don't promise filtering a provider can't do).

## Step 4: Run the actual setup

1. `docker compose up -d` -- starts Postgres plus the workers (and
   `eng-api`/`gitultra-mcp` on pro tier). The workers exiting
   immediately in the logs is expected (nothing queued yet), not a
   failure -- don't misinterpret `docker compose logs` showing an
   exited worker as broken.
2. Queue repos:
   ```
   docker compose run --rm --entrypoint python3 git-processor discover_repos.py <org> --provider <provider> [--config /var/lib/eng-metrics-suite/discover_config.yaml]
   ```
3. If change-failure-rate/MTTR was requested: help the user build
   `jira_project_map.yaml` (this maps Jira projects to repos -- there's
   no API that can derive this, it's a real hand-maintained mapping, so
   walk them through it rather than guessing), write it to
   `/var/lib/eng-metrics-suite/`, then:
   ```
   docker compose run --rm issue-processor python3 discover_jira_projects.py --config /var/lib/eng-metrics-suite/jira_project_map.yaml
   ```
4. If investment allocation/planning signals was requested: write
   `allocation_projects.yaml` (just a list of project keys, no repo
   mapping needed) to `/var/lib/eng-metrics-suite/`, then:
   ```
   docker compose run --rm issue-processor python3 discover_allocation_projects.py --config /var/lib/eng-metrics-suite/allocation_projects.yaml
   ```
5. Wait for initial import to make real progress before moving on --
   watch `docker compose logs -f git-processor pr-processor
   issue-processor` (Ctrl-C once you see real "imported commit"/"imported
   PR"-style lines, not just startup logs). For a large org this can
   take a while; don't block indefinitely -- check progress, and if it's
   clearly still working after a couple of minutes, tell the user it's
   running in the background and how to check back
   (`docker compose logs --tail 50 git-processor pr-processor`).

## Step 5: Connectivity/backend verification

Confirm the stack is actually healthy before declaring victory:
- `docker compose ps` -- every service should show `running` (or
  `exited (0)` for a worker that's caught up, which is healthy, not
  broken -- exit code non-zero is the actual problem signal).
- **Pro tier only**: confirm `eng-api` actually started successfully
  with the license key -- `docker compose logs eng-api | tail -20`
  should show a normal startup, not a license-check failure exiting
  the process. If it failed, the license key is the first thing to
  double check, not the rest of the stack.

## Step 6: Verify with a real report, then report done

This is the step that actually proves the install works -- don't skip
it or treat command-exit-code-zero as sufficient on its own.

```
docker compose run --rm eng-reports report.py --output /out/report.pdf
```
(add `--user "$(id -u):$(id -g)"` if the host user isn't UID 1000, check
with `id -u` first)

Then actually confirm the output, not just that the command exited
zero:
- The file exists at `./reports/report.pdf` and has a real, non-trivial
  size (a near-empty PDF suggests nothing actually imported yet, even
  if no error was thrown).
- If investment allocation/planning signals were configured, also run
  `allocation_report.py --output /out/allocation.pdf` and
  `planning_report.py --output /out/planning.pdf` and confirm both the
  same way.

**Only after confirming a real report exists with real content**,
report success to the user: what was imported (repo count if easily
available from the logs), which optional integrations are active, and
where the report(s) landed on disk. If verification fails at any point,
say so plainly and point at the specific failing step -- don't paper
over a failure with a generic "setup complete."
