# Blocking insecure code before it reaches production

## A self-hosted SonarQube quality gate on GitHub pull requests: the full setup, step by step

*Written by Riyasdeen J, DevOps Engineer. November 2026.*

---

## Why I built this

I wanted code quality and security to be checked on every change, automatically, as part of the normal pull request flow. A scan that runs only when someone remembers can't stop the next bad commit.

What I wanted:

- **Every pull request into the production branch is scanned automatically.** Nobody has to remember to run anything.
- **A failing scan blocks the merge.** "Please fix it later" doesn't work; a red check that won't let you merge does.
- **Only the code the PR adds is judged.** Old issues shouldn't block today's work, but new ones shouldn't get in.
- **The result shows up on the PR itself**, with a link to the exact line and an explanation of the problem.
- **It doesn't cost a licence**, and the code never leaves our own cloud account.

This document is the exact path I followed, including the mistakes I made on the way, so you can repeat it for any GitHub organisation.

---

## Contents

1. [What you end up with](#1-what-you-end-up-with)
2. [Choosing the SonarQube edition](#2-choosing-the-sonarqube-edition)
3. [Before you start](#3-before-you-start)
4. [Step 1: The SonarQube server](#step-1-the-sonarqube-server)
5. [Step 2: Expose it safely (load balancer and DNS)](#step-2-expose-it-safely)
6. [Step 3: First login and global settings](#step-3-first-login-and-global-settings)
7. [Step 4: Create the GitHub App](#step-4-create-the-github-app)
8. [Step 5: Connect SonarQube to GitHub](#step-5-connect-sonarqube-to-github)
9. [Step 6: Import the repository as a project](#step-6-import-the-repository)
10. [Step 7: The quality gate](#step-7-the-quality-gate)
11. [Step 8: A CI user and token](#step-8-a-ci-user-and-token)
12. [Step 9: Repository secret and variable](#step-9-repository-secret-and-variable)
13. [Step 10: The workflow and project files](#step-10-the-workflow-and-project-files)
14. [Step 11: Make the check required](#step-11-make-the-check-required)
15. [Step 12: Prove it works with a test PR](#step-12-prove-it-works)
16. [Step 13: Users, groups and permissions](#step-13-users-groups-and-permissions)
17. [Step 14: Log in with your GitHub organisation](#step-14-log-in-with-your-github-organisation)
18. [Step 15: Add another repository](#step-15-add-another-repository)
19. [How developers use it day to day](#how-developers-use-it-day-to-day)
20. [Troubleshooting: every error I actually hit](#troubleshooting)
21. [Security notes](#security-notes)
22. [Maintenance](#maintenance)
23. [What's next: future enhancements](#whats-next)

---

## 1. What you end up with

```
 Developer opens a PR into main (from any branch)
            │
            ▼
 GitHub Actions starts the "SonarQube" job on a self-hosted runner
            │   the scanner sends the code to SonarQube
            ▼
 SonarQube (Community Build + branch plugin) on your own server
   checks only the PR's new / changed code against the quality gate
            │
            ├── passes ──► "SonarQube" check is green ──► PR can be merged
            │
            └── fails  ──► "SonarQube" check is red  ──► merge is blocked
                           + a check run on the PR lists the problems,
                             with a link to the exact line in SonarQube

 After the merge, the push to main is scanned too.
 That becomes the baseline the next PR is compared against.
```

The pieces:

| Piece | What it does |
|---|---|
| SonarQube server (Docker on a VM) | Runs the analysis and stores the results |
| PostgreSQL (same Docker host) | SonarQube's database |
| Load balancer + DNS | Gives the server an HTTPS address, reachable only from allowed IPs |
| GitHub App | Lets SonarQube read the repo list, post results on pull requests, and log people in with their GitHub organisation account |
| GitHub Actions workflow | Runs the scanner on every PR and push |
| Branch ruleset | Makes the "SonarQube" check required, so a red check blocks the merge |
| Groups and permissions | Developers can read results; only leads can accept an issue or mark a hotspot safe |

Steps 1–12 set this up for one repository. Steps 13–15 turn it into an organisation setup: permissions, GitHub login, and the checklist for every further repository.

---

## 2. Choosing the SonarQube edition

This is the most important decision, because the free edition has one big limitation.

| Option | PR analysis | Cost | Notes |
|---|---|---|---|
| SonarQube **Community Build**, plain | **No** | Free | Analyses one branch only. Every PR scan overwrites the previous one, so "new code" and PR comments don't work. |
| SonarQube Community Build **+ community branch plugin** | Yes | Free | The unofficial [mc1arke/sonarqube-community-branch-plugin](https://github.com/mc1arke/sonarqube-community-branch-plugin) adds branch and PR analysis. Not supported by SonarSource. |
| SonarQube Server **Developer Edition** | Yes | Paid (licence by lines of code) | Official PR analysis and support. |
| **SonarQube Cloud** (SaaS) | Yes | Paid for private repos | Nothing to host, but your code is uploaded to their cloud. |

**I chose Community Build + the branch plugin**: free, self-hosted, full PR analysis.

The trade-off is that the plugin version **must match** the SonarQube version exactly (plugin `26.10.0` works with SonarQube `26.10`). Plugin releases sometimes arrive weeks after a SonarQube release, so you upgrade on the plugin's schedule, not SonarSource's. If you have budget, Developer Edition removes that worry and the rest of this guide stays almost the same.

---

## 3. Before you start

You need:

- **A Linux VM**: 2 vCPU and 8 GB RAM minimum (I used an AWS `t3.large`), 40 GB disk, in a private subnet with outbound internet through a NAT.
- **A way to publish it over HTTPS**: a load balancer with a TLS certificate (or a reverse proxy), plus a DNS name such as `sonarqube.example.com`.
- **Owner or admin rights in the GitHub organisation**, to create a GitHub App and to edit branch rulesets.
- **A GitHub Actions runner that can reach the server.** If your org has an IP allow list, GitHub-hosted runners are blocked, so you need a **self-hosted runner with a static IP**. We already had one.
- About half a day.

Placeholders used in this guide (replace with yours):

| Placeholder | Meaning |
|---|---|
| `sonarqube.example.com` | Your SonarQube address |
| `your-org` | Your GitHub organisation |
| `your-app` | The repository you're protecting |
| `main` | The production branch (ours is `master`) |
| `<NAT_IP>` | The public IP the SonarQube server uses for outbound traffic |
| `<RUNNER_IP>` | The public IP of your self-hosted runner |

---

## Step 1: The SonarQube server

### 1.1 Prepare the VM (Ubuntu)

```bash
sudo apt-get update
sudo apt-get install -y docker.io docker-compose-v2
sudo systemctl enable --now docker
docker compose version          # must print a version
```

> **Gotcha I hit:** on a fresh Ubuntu box, `docker compose up -d` failed with
> `unknown shorthand flag: 'd' in -d`. That means the Compose plugin isn't installed;
> Docker doesn't know `compose` and complains about `-d` instead. Install `docker-compose-v2`.

SonarQube embeds Elasticsearch, which refuses to start without these kernel settings:

```bash
printf 'vm.max_map_count=524288\nfs.file-max=131072\n' | sudo tee /etc/sysctl.d/99-sonarqube.conf
sudo sysctl --system
```

### 1.2 docker-compose.yml

Create a folder (I used `/opt/sonarqube`) and put this file in it:

```yaml
services:
  sonarqube:
    # Prebuilt image: SonarQube Community Build with the branch plugin.
    # The tag = the SonarQube version; plugin and SonarQube must always match.
    image: mc1arke/sonarqube-with-community-branch-plugin:26.10.0.133078-community
    container_name: sonarqube
    restart: unless-stopped
    depends_on:
      db:
        condition: service_healthy
    environment:
      SONAR_JDBC_URL: jdbc:postgresql://db:5432/sonar
      SONAR_JDBC_USERNAME: sonar
      SONAR_JDBC_PASSWORD: ${POSTGRES_PASSWORD:?set POSTGRES_PASSWORD in .env}
      # Don't set SONAR_WEB_JAVAADDITIONALOPTS / SONAR_CE_JAVAADDITIONALOPTS:
      # the image uses them to load the plugin.
    ports:
      - "9000:9000"          # only the load balancer may reach this (security group)
    volumes:
      - sonarqube_data:/opt/sonarqube/data
      - sonarqube_extensions:/opt/sonarqube/extensions
      - sonarqube_logs:/opt/sonarqube/logs
    ulimits:
      nofile: { soft: 131072, hard: 131072 }
      nproc: { soft: 8192, hard: 8192 }

  db:
    image: postgres:17-alpine
    container_name: sonarqube-db
    restart: unless-stopped
    environment:
      POSTGRES_USER: sonar
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:?set POSTGRES_PASSWORD in .env}
      POSTGRES_DB: sonar
    volumes:
      - postgresql:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U sonar -d sonar"]
      interval: 10s
      timeout: 5s
      retries: 10

volumes:
  sonarqube_data:
  sonarqube_extensions:
  sonarqube_logs:
  postgresql:
```

### 1.3 Create the database password once, then start

```bash
cd /opt/sonarqube
# Only if .env doesn't exist yet:
[ -f .env ] || { printf 'POSTGRES_PASSWORD=%s\n' "$(openssl rand -hex 24)" > .env && chmod 600 .env; }

docker compose up -d
docker compose logs -f sonarqube     # wait for "SonarQube is operational", then Ctrl+C
curl -s localhost:9000/api/system/status
# {"id":"...","version":"26.10.0.133078","status":"UP"}
```

> **Gotcha I hit (the big one):** I generated `.env` a second time after the database
> already existed. Postgres only reads `POSTGRES_PASSWORD` when it first creates the
> database, so SonarQube kept failing with
> `FATAL: password authentication failed for user "sonar"` and restarted in a loop.
> The guard above (`[ -f .env ] ||`) prevents it. To fix it without losing data:
>
> ```bash
> PW=$(grep '^POSTGRES_PASSWORD=' .env | cut -d= -f2)
> docker compose exec -T db psql -U sonar -d sonar -c "ALTER USER sonar WITH PASSWORD '$PW';"
> docker compose restart sonarqube
> ```

> **Not stuck, just following:** `docker compose logs -f` never exits on its own. When you see
> `SonarQube is operational`, press Ctrl+C. The server keeps running.

---

## Step 2: Expose it safely

I put SonarQube behind our existing load balancer instead of giving the VM a public IP.

1. **Security group on the VM:** allow TCP **9000** only from the load balancer's security group. Nothing else inbound.
2. **Target group:** HTTP, port 9000, health check path `/api/system/status`, success code `200`. Register the VM.
3. **Listener rule** on the HTTPS (443) listener: host header `sonarqube.example.com` → forward to the target group.
4. **Who can reach it:** our load balancer already only allowed office IPs on 443. Make sure **`<RUNNER_IP>`** (your self-hosted runner) is allowed too, or the scan can't upload its results.
5. **DNS:** a CNAME `sonarqube` → the load balancer's DNS name.

> **If you use Cloudflare:** set the record to **DNS only (grey cloud)**. If it's proxied, the
> load balancer sees Cloudflare's IPs instead of your office's, and an IP-based rule blocks everyone.

Check from an allowed network: `https://sonarqube.example.com` should show the login page.

---

## Step 3: First login and global settings

1. Log in with **`admin` / `admin`** and **change the password straight away.** Until you do, anyone who can reach the page can log in.
2. A page **"Installation of plugins"** appears: click **I understand the risk**. It's the expected warning for the branch plugin.
3. **Administration → Configuration → General Settings:**
   - **General → Server base URL** = `https://sonarqube.example.com`. Without it, the links SonarQube posts on PRs point to `http://localhost:9000`, which I only noticed after the first PR.
   - **Pull Request → Images base URL** = `https://raw.githubusercontent.com/mc1arke/sonarqube-community-branch-plugin/master/src/main/resources/static`. GitHub can't load badge images from an IP-restricted server. (Opening that URL in a browser gives 404, because it's a folder. That's expected.)
   - **Security → Force user authentication** = on.
4. **Administration → Projects → Management → Default visibility** = **Private**.

---

## Step 4: Create the GitHub App

SonarQube uses a GitHub App to list repositories and to post the result on each PR.

**Org settings → Developer settings → GitHub Apps → New GitHub App**

| Field | Value |
|---|---|
| GitHub App name | e.g. `SonarQube YourOrg` (must be unique on GitHub) |
| Homepage URL | `https://sonarqube.example.com` |
| Callback / Redirect URIs | **three entries** (see below) |
| Webhook → Active | **unticked** |
| Where can it be installed? | Only on this account |

**Redirect URIs.** Add all three:

```
https://sonarqube.example.com/oauth2/callback/github
https://sonarqube.example.com
https://sonarqube.example.com/projects/create
```

> **Gotcha I hit:** without the exact `/projects/create` entry, "Import from GitHub" fails on
> GitHub with **"Be careful! The redirect_uri is not associated with this application."**
> GitHub Apps match redirect URIs exactly unless "Allow wildcard matching" is on. Listing
> the exact path is safer than turning wildcards on.

**Repository permissions:**

| Permission | Access |
|---|---|
| Administration | Read-only |
| Checks | Read and write |
| Contents | Read-only |
| Metadata | Read-only (mandatory) |
| Pull requests | Read and write |

Organisation permissions: none needed.

After **Create GitHub App**:

1. Note the **App ID** and **Client ID** (top of the General page).
2. **Credentials → Client secrets → Generate a new client secret.** Copy it now; it's shown once.
3. **Credentials → Private keys → Generate a private key.** A `.pem` file downloads.
4. **Install App → Install** on your org → **Only select repositories** → pick `your-app`.

### The IP allow list (if your org has one)

**Org settings → Authentication security → IP allow list → Add** `<NAT_IP>/32`, the address your SonarQube server uses for outbound traffic.

> **Gotcha I hit:** the configuration check failed with "Authentication failed, verify the
> Client Id, Client Secret and Private Key". The keys were fine. The real reason was in
> SonarQube's own log:
>
> ```bash
> docker compose exec sonarqube grep -iE "github|401|403" /opt/sonarqube/logs/web.log | tail
> ```
>
> `403 … the organization has an IP allow list enabled, and your IP address is not permitted`.
> Find the server's outbound IP with `curl -s https://checkip.amazonaws.com` on the server.
> If the org page doesn't let you add it, the list is managed at the **enterprise** level.

---

## Step 5: Connect SonarQube to GitHub

**Projects → Create project → Import from GitHub → Setup** (or Administration → DevOps Platform Integrations → GitHub):

| Field | Value |
|---|---|
| Configuration name | `github` |
| GitHub API URL | `https://api.github.com/` |
| GitHub App ID | from step 4 |
| Client ID | from step 4 (copy it; `l` and `I` look the same) |
| Client Secret | from step 4 |
| Private Key | the whole `.pem`, including the `-----BEGIN…` and `-----END…` lines |

Save, then **Check configuration**: it must be green. Delete the `.pem` from your Downloads folder afterwards.

---

## Step 6: Import the repository

1. **Projects → Create project → Import from GitHub.** Authorise the app when GitHub asks, pick your org, tick `your-app` → **Import**.
2. **New code definition:** choose **Follows the instance's default**. For PRs, SonarQube always treats the PR's own changes as new code, so this setting only affects the branch overview page.
3. **Fix the project key.** The import creates a long key like `your-org_your-app_9c15d84a-…`. I changed it to something short and fixed, so it matches the file in the repo:
   **Project settings → Update key** → `your-app`.

> Do the key change **before** the first scan. A scan with a key that doesn't exist creates
> a second, empty project.

---

## Step 7: The quality gate

The built-in **Sonar way** gate requires 80% test coverage on new code. Our repo had no test coverage report yet, so every PR would have failed. I made a copy without that condition:

1. **Quality Gates → Sonar way → Copy** → name it `your-app`.
2. In the copy, **delete the Coverage condition.** I kept:
   - Issues on new code: greater than 0 → fail
   - Security Hotspots Reviewed: less than 100% → fail
   - Duplicated Lines (%) on new code: greater than 3% → fail
3. Don't click **"Review and update"** in the blue banner. It adds the recommended conditions back, Coverage included.
4. At the bottom of the gate page: **Projects → Without** → tick `your-app`. It's assigned immediately.

When you add test coverage later, put the Coverage condition back (see [What's next](#whats-next)).

---

## Step 8: A CI user and token

Don't let the pipeline use a person's token. When that person leaves, the pipeline breaks.

1. **Administration → Security → Users → Create User:** `ci-github`, long random password.
2. **Project → Project settings → Permissions:** give `ci-github` **only "Execute Analysis"**.
3. Log in as `ci-github` → **My Account → Security → Generate Tokens:**
   type **Project Analysis Token**, project `your-app`, expiry 1 year. Copy it.

A project analysis token can analyse that one project and nothing else, so if it leaks the damage is small. Put a calendar reminder to rotate it before it expires.

---

## Step 9: Repository secret and variable

**Repo → Settings → Secrets and variables → Actions:**

| Tab | Type | Name | Value |
|---|---|---|---|
| Secrets | **Repository** secret | `SONAR_TOKEN` | the token from step 8 |
| Variables | **Repository** variable | `SONAR_HOST_URL` | `https://sonarqube.example.com` |

> Use **repository** secrets, not **environment** secrets. The job doesn't declare an
> `environment:`, so environment secrets are invisible to it and the scan fails with
> "Running this GitHub Action without SONAR_TOKEN".

> **Gotcha I hit:** our first PR scans ran *before* the token existed, failed after 15 seconds,
> and looked like a broken setup. Always add the secret first, then open the PR.

---

## Step 10: The workflow and project files

Two files go into the repository.

### `sonar-project.properties` (repo root)

```properties
sonar.projectKey=your-app
sonar.projectName=your-app

sonar.sources=src
sonar.tests=tests
sonar.sourceEncoding=UTF-8

# Not application code
sonar.exclusions=**/*.md,**/*.css,**/*.ico,**/next-env.d.ts

# When you add tests with coverage, point SonarQube at the report:
# sonar.javascript.lcov.reportPaths=coverage/lcov.info
```

### `.github/workflows/sonarqube.yml`

```yaml
name: SonarQube

on:
  pull_request:
    branches: [main]          # every PR into main, from any branch
  push:
    branches: [main]          # the baseline PRs are compared to
  merge_group:                # if main uses a merge queue
  workflow_dispatch:

permissions:
  contents: read

concurrency:
  group: sonarqube-${{ github.event.pull_request.number || github.ref }}
  cancel-in-progress: true

jobs:
  sonarqube:
    name: SonarQube            # this name is the required status check
    runs-on: [self-hosted, linux]   # static-IP runner that can reach the server
    timeout-minutes: 20
    steps:
      # In the merge queue the PR was already gated: report the check, don't scan again.
      - name: Merge queue (already gated on the PR)
        if: github.event_name == 'merge_group'
        run: echo "The quality gate ran on the pull request."

      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
        if: github.event_name != 'merge_group'
        with:
          fetch-depth: 0               # full history: SonarQube uses git blame for "new code"
          persist-credentials: false

      - uses: actions/setup-node@820762786026740c76f36085b0efc47a31fe5020 # v7.0.0
        if: github.event_name != 'merge_group'
        with:
          node-version: "22"           # the JS/TS analyser runs on Node

      - name: SonarQube scan
        if: github.event_name != 'merge_group'
        uses: SonarSource/sonarqube-scan-action@d209202bc7d53ff1cc128f7f907dac145c9d6ae9 # v8.3.0
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
          SONAR_HOST_URL: ${{ vars.SONAR_HOST_URL }}
        with:
          # PRs wait for the gate and fail the check when it's red.
          # Pushes only update the baseline. scm.revision points the PR result at the PR head.
          args: >
            -Dsonar.qualitygate.wait=${{ github.event_name == 'pull_request' }}
            -Dsonar.qualitygate.timeout=600
            ${{ github.event_name == 'pull_request' && format('-Dsonar.scm.revision={0}', github.event.pull_request.head.sha) || '' }}
```

Notes:

- **Actions are pinned to commit SHAs**, not tags, so a moved tag can't change what runs. Look up the current SHAs and let Dependabot keep them fresh.
- The scanner exits with **code 3** when the quality gate fails. That's the "red check" you want.
- PR key, branch and base are read automatically from the GitHub Actions environment. You don't need `sonar.pullrequest.*` arguments.
- Want to gate a pre-production branch too? Add it to both `branches:` lists.

---

## Step 11: Make the check required

Until this step, a red check is only a warning.

**Repo → Settings → Rules → Rulesets** → the ruleset for `main` → **Add rule → Require status checks to pass → Add checks** → `SonarQube`, source **GitHub Actions** → Save.

> **Gotchas I hit:**
>
> - **Check your rulesets are actually active.** A ruleset in the *Disabled* state looks
>   configured but enforces nothing, so a PR with red checks can still be merged.
> - **Merge queue:** if `main` uses a merge queue, the workflow must have the `merge_group`
>   trigger (above) *before* you make the check required. Otherwise queue entries wait
>   forever for a check that never runs.
> - **The workflow must exist on `main` first.** Add the requirement after the workflow
>   file has been merged into `main`; otherwise PRs from branches without the file can
>   never pass.
> - Pick the check named **`SonarQube`** (from GitHub Actions), not "SonarQube Code
>   Analysis" (the check run posted by the GitHub App).

---

## Step 12: Prove it works

Don't trust a setup you haven't seen fail. Open a **draft** pull request with code that is meant to fail:

```ts
// src/lib/sonar-gate-test.ts  (TEMPORARY, never merge)
import { execSync } from "child_process";

const password = "sonar-gate-test-only";          // hard-coded password

export function listFiles(input: string): string {
  return execSync(`ls ${input}`).toString();       // shell command built from input
}

export function isMissing(value: number): boolean {
  return value === NaN;                            // always false: a bug
}

export const configuredPassword = password;
```

Expected result:

- The **SonarQube** check goes red; the job log ends with `QUALITY GATE STATUS: FAILED`.
- A **SonarQube Code Analysis** check run appears on the PR: *"Quality Gate failed: 2 New Issues"*, with a link.
- SonarQube shows the PR under **Pull Requests** with the issues listed.

Then **close the PR without merging** and delete the branch.

> **Always use a draft PR for tests.** A normal test PR can be merged by accident, for example
> during a routine clean-up of open PRs, and the deliberately broken file then lands in a real
> branch. GitHub won't let anyone merge a draft.

Also check a clean PR goes **green**. Ours (a revert that only deleted code) passed the gate and showed the green check run.

---

## Step 13: Users, groups and permissions

A quality gate is only as strong as the people who can override it. In SonarQube, anyone with **Administer Issues** can mark an issue *Accepted* or *False positive*, and anyone with **Administer Security Hotspots** can mark a hotspot *Safe*. Either way it stops counting, and a red PR turns green without a single line of code changing.

> **Gotcha I hit:** out of the box, the `sonar-users` group, which means *every person who
> logs in*, had both of those permissions on our projects. Any developer could have passed
> their own gate. Check this before you let more people in (step 14).

### Who gets what

| Who | Group / user | Purpose |
|---|---|---|
| SonarQube admins | `sonar-administrators` (built in) | Manage the server and projects |
| Everyone who logs in | `sonar-users` (built in) | Read results and code |
| Team leads | `tech-leads` (you create it) | Accept issues, review security hotspots |
| The pipeline | `ci-github` (local user, step 8) | Run scans, nothing else |

Permissions on each project:

| | Browse | See Source Code | Administer Issues | Administer Security Hotspots | Administer | Execute Analysis |
|---|---|---|---|---|---|---|
| `sonar-administrators` | ✅ | ✅ | | | ✅ | |
| `sonar-users` | ✅ | ✅ | ❌ | ❌ | | |
| `tech-leads` | ✅ | ✅ | ✅ | ✅ | | |
| `ci-github` | ✅ | | | | | ✅ |

`ci-github` needs **Browse** as well as Execute Analysis: the workflow reads the quality gate result back, and that's a read.

### Set it up

1. **Create the leads group:** **Administration → Security → Groups → Create Group** → `tech-leads`. Add members once they have logged in (after step 14 a user appears on their first GitHub login).
2. **Fix each existing project:** **Project → Project settings → Permissions**:
   - **Visibility → Private.** A *Public* project can be browsed without these permissions.
   - Set the table above: untick Administer Issues and Administer Security Hotspots for `sonar-users`; add `tech-leads` and `ci-github`.
3. **Make new projects start right:** **Administration → Security → Permission Templates → Default template** → the same table. Every project imported later gets it automatically.
4. **Private by default:** **Administration → Projects → Management → Default visibility of new projects → Private.**

> **Gotcha I hit:** a project imported from GitHub showed **Public** even though I thought I had
> set the default to private. After every import, look at the badge next to the project name.

---

## Step 14: Log in with your GitHub organisation

A shared login tells you nothing about who accepted an issue. With GitHub login, everyone signs in with their own organisation account, and leaving the organisation means losing SonarQube access too. You reuse the **same GitHub App** from step 4.

If your GitHub organisation signs people in through a company identity provider (Microsoft Entra ID / Azure AD, Okta and so on, called *Enterprise Managed Users*), the chain becomes:

```
SonarQube "Log in with GitHub" → GitHub → company sign-in (with MFA) → back to SonarQube
```

**Do step 13 first.** The moment GitHub login is on, every member of the organisation can log in and lands in `sonar-users`.

### 14.1 Give the GitHub App two more permissions

The permissions are edited on the **app's own settings page**: org **Settings → Developer settings → GitHub Apps → your app → Edit → Permissions & events**. The installation page (org Settings → GitHub Apps → Configure) only controls which repositories the app can use.

| Section | Permission | Access |
|---|---|---|
| Organization permissions | Members | Read-only (to check the user belongs to your org) |
| Account permissions | Email addresses | Read-only |

While you're there, set anything SonarQube doesn't need back to **No access**. I had picked *Organization → Administration* by mistake earlier.

**Save changes**, then an org owner must **accept** them: org **Settings → GitHub Apps → your app** → the banner about new permissions → **Accept new permissions**. Until then nothing changes.

The callback URL SonarQube needs, `https://sonarqube.example.com/oauth2/callback/github`, is already on the app from step 4.

### 14.2 Turn on GitHub login in SonarQube

**Administration → Configuration → General Settings → Authentication → GitHub** → **Configure manually**. Don't use "Create GitHub App": you already have one.

| Field | Value |
|---|---|
| Client ID | from the app's General page |
| Client Secret | the app's **Credentials → Client secrets**. You can't view an old one again, so generate a new one; the old one keeps working for the DevOps integration |
| GitHub App ID | from the app's General page |
| Private key | the whole `.pem` (generate a new key if you deleted the file) |
| GitHub URL / API URL | `https://github.com/` and `https://api.github.com/` |
| Organizations | `your-org`: only members can log in |
| Allow users to sign up | On: a user is created on their first login |
| Synchronize teams as groups (if shown) | Optional: GitHub teams become SonarQube groups |

**Save configuration**, then **Enable**. If asked how to provision users, choose **just-in-time** (on first login).

### 14.3 Test it

Open SonarQube in a private window → **Log in with GitHub** → **Authorize** the app on first use → you land in SonarQube as yourself. Check you can see the projects but can't accept issues.

> **Gotcha I hit:** in the private window GitHub's password form said
> **"Incorrect username or password"**. With Enterprise Managed Users there *is* no GitHub
> password: you sign in through the company identity provider. Either test in a browser
> that's already signed in to GitHub, or start the sign-in from your enterprise's SSO page
> first. Normal users never see this, because they're already signed in to GitHub.

### 14.4 Keep these local

- **`admin`:** your way in if GitHub or the identity provider is down. Strong password, not shared.
- **`ci-github`:** the pipeline uses its token; it never logs in through GitHub.
- **Old local accounts:** once someone has logged in with GitHub, deactivate their local account (**Administration → Security → Users**), so each person has one account.

---

## Step 15: Add another repository

The server, the GitHub App, the IP allow list, the quality gate and GitHub login are shared. Each further repository takes about fifteen minutes. This is the checklist I use.

| # | Where | Do |
|---|---|---|
| 1 | GitHub org | **Settings → GitHub Apps → your app → Configure → Repository access** → add the repo → Save |
| 2 | SonarQube | **Projects → Create project → Import from GitHub** → tick the repo → Import → New code: *Follows the instance's default* → Create |
| 3 | SonarQube | **Project settings → Update key** → a short key, e.g. `your-other-app` (must match the repo file) |
| 4 | SonarQube | **Project settings → Permissions** → Private + the step 13 table |
| 5 | SonarQube | **Quality Gates** → your gate → **Projects → Without** → tick the project |
| 6 | SonarQube (as `ci-github`) | **My Account → Security** → Project Analysis Token for this project |
| 7 | GitHub repo | **Repository** secret `SONAR_TOKEN`, **repository** variable `SONAR_HOST_URL` |
| 8 | Git | The two files from step 10, on a branch from **every long-lived base branch** (e.g. `main` and `staging`), identical on each |
| 9 | GitHub repo | Open the PRs, **only after step 7**. Expect SonarQube ✅ |
| 10 | GitHub repo | Merge them. The push to `main` scans the main branch |
| 11 | GitHub repo | Ruleset: require `SonarQube`, **after** the workflow is on every base branch |
| 12 | GitHub repo | A draft PR with a deliberate issue goes red → close it |

Two tips save time:

- **Skip the "Analysis Method" page** that appears after an import (and the **Configure analysis** link on the project card). It's a tutorial for writing a workflow; you already have one. It disappears after the first scan.
- **Set your gate as the default once** (**Quality Gates → your gate → ⋮ → Set as Default**), and new projects get it without step 5.

### When the app is in a subfolder

`sonar-project.properties` for a repo whose app lives in `src/`, with its own `package.json` and tests:

```properties
sonar.projectKey=your-other-app
sonar.projectName=your-other-app
sonar.sourceEncoding=UTF-8

sonar.sources=src
sonar.tests=src/tests
sonar.test.inclusions=src/tests/**
sonar.exclusions=src/tests/**,src/public/**,src/docs/**,**/*.md,**/*.css,**/*.d.ts
sonar.typescript.tsconfigPaths=src/tsconfig.json
```

Tests are excluded from `sonar.sources` and listed under `sonar.tests`, because a file can't be both.

> **Gotchas I hit while adding more repositories:**
>
> - **The required check waited forever:** "SonarQube — Expected — Waiting for status to be
>   reported". GitHub only runs workflows found in the PR's code merged with its base branch.
>   The rule was required on `main` before the workflow file was on `main`, so PRs from
>   branches cut from `main` never started the job. Put the file on every base branch
>   **first**, then require the check. Stuck PRs start the job after **Update branch**.
> - **The PRs were opened before the token existed** and SonarQube failed in seconds. You
>   don't need new PRs: **re-run the failed job** (Checks → SonarQube → Re-run).
> - **"Project's Main Branch is not analyzed yet"** stays until the workflow is on `main` and
>   something is pushed there. Merging the PR does that; **Actions → SonarQube → Run
>   workflow → `main`** also works once the file is on `main`.
> - **A project card showed the wrong branch.** SonarQube uses GitHub's default branch as the
>   main branch at import. If you renamed it, rename it back (branch selector → **Manage
>   branches**) so the card shows `main`.
> - **Other red checks on the same PR weren't SonarQube.** A repository's existing checks can
>   fail for their own reasons: missing CI variables, a cloud role that doesn't trust the new
>   repo yet, an outdated test. Read which check is red before you debug SonarQube.

---

## How developers use it day to day

**Nothing to start by hand.** Open a PR into `main` and the scan runs. Push again and it runs again.

**Logging in:** open `https://sonarqube.example.com` → **Log in with GitHub**. Your organisation account is enough; there's no separate SonarQube password.

When the check is **red**:

1. On the PR, open **Checks → SonarQube Code Analysis → View in SonarQube.**
2. **Overview** shows which gate condition failed.
3. **Issues:** each finding with file, line, severity, and "Why is this an issue?" / "How can I fix it?".
4. **Security Hotspots:** code that needs a security review. Fix it, or ask a lead to review it and mark it **Safe** with a reason (developers can't do that themselves; see step 13).
5. Fix, push. The check updates by itself.

To see **everything** (not just one PR): log in → **Projects → your-app** → branch `main` → **Issues**, filtered by **Severity** or by **Directory** (for example your own service folder).

Only **new code** blocks a PR. Existing issues are listed but don't block anyone, which lets you clean up the backlog gradually without stopping work.

---

## Troubleshooting

Every one of these happened to me during this setup.

| Symptom | Cause | Fix |
|---|---|---|
| `unknown shorthand flag: 'd' in -d` | Docker Compose plugin missing | `apt-get install -y docker-compose-v2` |
| SonarQube restarts, `max virtual memory areas vm.max_map_count [65530] is too low` | Kernel setting missing | Step 1.1 `sysctl` |
| Restart loop, `password authentication failed for user "sonar"` | `.env` regenerated after the DB was created | `ALTER USER` command in step 1.3, or `docker compose down -v` on a fresh install |
| `docker compose logs -f` "hangs" | It follows the log forever | Ctrl+C; check `curl localhost:9000/api/system/status` |
| Images base URL gives 404 in the browser | It's a folder; GitHub doesn't list folders | Expected. A file like `.../common/bug.png` returns 200 |
| GitHub config: "Authentication failed, verify the Client Id…" | Usually the org **IP allow list** (check `web.log`) | Add the server's outbound NAT IP |
| "The redirect_uri is not associated with this application" | Exact redirect URI missing | Add `https://<host>/projects/create` to the app |
| PR check fails in 15 s: "without SONAR_TOKEN" | Secret missing, or saved as an *environment* secret | Repository secret `SONAR_TOKEN` |
| Workflow can't reach the server / timeout | Runner IP not allowed on the load balancer | Allow `<RUNNER_IP>` |
| Links on the PR open `localhost:9000` | Server base URL not set | Step 3 |
| A PR with a red check got merged | Check not required, or the ruleset is disabled | Step 11 |
| Merge queue waits forever | Required check doesn't run on `merge_group` | Add the `merge_group` trigger |
| A second empty project appears | Scan used a key that doesn't match the project | Update the project key before the first scan |
| SonarQube log: `flood stage disk watermark` | Disk nearly full | Grow the volume (40 GB+) |
| Required check stuck on "Expected — Waiting for status to be reported" | The workflow file isn't on the PR's base branch | Merge the workflow into every base branch first, then **Update branch** on stuck PRs (step 15) |
| New PR fails in seconds although `SONAR_TOKEN` now exists | The PR was opened before the secret was added | Re-run the failed job; no need for a new PR |
| "Project's Main Branch is not analyzed yet" | Nothing has been pushed to `main` with the workflow on it | Merge the PR into `main`, or **Run workflow** on `main` |
| Project card shows another branch, not `main` | The main branch was renamed in SonarQube | Rename it back (branch selector → Manage branches) |
| An imported project shows **Public** | Visibility not set for this project | **Project settings → Permissions → Private** |
| GitHub login: "Incorrect username or password" | Enterprise Managed Users have no GitHub password | Sign in through the company identity provider (step 14.3) |
| GitHub login doesn't let members in | App lacks Members / Email permissions, or the change wasn't accepted | Step 14.1, then **Accept new permissions** |

The most useful command during setup:

```bash
docker compose logs --tail=500 sonarqube | grep -E "ERROR|WARN|Caused by|FATAL" | tail -30
```

The `Caused by:` line nearest the bottom is usually the real reason.

---

## Security notes

- **Give the VM its own IAM role** with only what it needs (for example SSM for shell access). Don't reuse an application's role. SonarQube runs a third-party plugin and should not be able to read your application secrets.
- **The token is scoped.** A project analysis token from a CI-only user can analyse one project, nothing else.
- **Only leads can override the gate.** Developers can read issues and code, but accepting an issue or marking a hotspot safe is limited to a leads group (step 13).
- **Login follows your organisation.** With GitHub login limited to your org (step 14), removing someone from the org, or from the company identity provider, removes their SonarQube access too.
- **Keep SonarQube off the open internet.** Behind the load balancer, only office and runner IPs can reach it, and login is forced.
- **The NAT IP you add to GitHub's allow list may be shared** by other servers in the same subnets. It's usually acceptable because they still need valid credentials, but say so to whoever approves the change.
- **Never put a real secret in a test PR.** Use an obviously fake value, as in the example above.
- **Pin the Docker image tag and the Actions SHAs.** Upgrades happen on purpose, not by accident.

---

## Maintenance

**Backups:** daily snapshot of the VM's disk (AWS Data Lifecycle Manager or AWS Backup), keep 7.

**Upgrading SonarQube:**

1. Check that a matching tag exists for `mc1arke/sonarqube-with-community-branch-plugin`.
2. Take a snapshot.
3. Change the tag in `docker-compose.yml`, then `docker compose pull && docker compose up -d`.
4. If the UI asks for a database upgrade, open `https://sonarqube.example.com/setup`.

The image copies the plugin jar into place on every start, so a persistent `extensions` volume doesn't keep an old plugin around.

**Token rotation:** before the yearly expiry, generate a new token for `ci-github` and replace `SONAR_TOKEN`.

**Telemetry (optional):** to stop SonarQube sending anonymous usage statistics, add `SONAR_TELEMETRY_ENABLE: "false"` under `environment:`.

---

## What's next

Done so far: the PR quality gate, GitHub organisation login, a permission model where only leads can override the gate, and the same setup on three repositories. Here's my roadmap.

### Roll out to every repository

- **Reusable workflow.** Move the job into one central repo as a `workflow_call` workflow, so each repo only has a few lines and fixes happen in one place.
- **Org-level ruleset.** Require the `SonarQube` check from an **organisation ruleset** that targets repositories by name pattern or custom property, instead of editing every repo.
- **Organisation-level secret and variable.** One `SONAR_HOST_URL` variable and one token secret shared with selected repositories, instead of setting them per repo. That needs a global analysis token, which can scan every project, so weigh it against per-project tokens.

### Make the gate smarter

- **Test coverage.** Add a test runner with coverage (for a TypeScript app, Vitest with `lcov`), point `sonar.javascript.lcov.reportPaths` at it, and bring back the Coverage condition, starting low (say 50% on new code) and raising it over time.
- **Tuned quality profiles.** Review which rules fire most, turn off the noisy ones that don't apply to your stack, and raise the severity of the ones that matter to you.
- **A hotspot review routine.** Agree who reviews security hotspots and how fast, so the gate never blocks for days on a review.

### Improve the developer experience

- **SonarQube for IDE (SonarLint) in connected mode,** so developers see the same rules in VS Code before they even push.
- **GitHub teams as SonarQube groups,** so a lead added to the right GitHub team gets the lead permissions automatically.
- **Chat notifications:** a webhook from SonarQube or a small workflow step that posts failed gates to Slack, Teams or Mattermost.
- **A weekly summary** of new, fixed and open issues per repository for the team leads.

### Harden the platform

- **Infrastructure as code:** turn the console steps (VM, security groups, load balancer rule, IAM role) into CloudFormation or Terraform.
- **Managed database:** move PostgreSQL to RDS for automated backups and easier upgrades.
- **Private access only:** serve SonarQube over VPN or a private load balancer instead of an IP-restricted public one.
- **Monitoring:** alarm on the target group's health and on disk usage.

### Combine with other scanners

SonarQube is one layer. In the same PR checks we also run **secret scanning (gitleaks)**, **SAST rules (Semgrep)**, **dependency audit (npm audit)** and **Dockerfile / IaC scanning (hadolint, Trivy)**. Next I want **container image scanning** before deploy and **SBOM generation** per release.

### When to move to a paid edition

If you depend heavily on PR analysis across many repos, or the plugin lags too far behind SonarQube releases, **Developer Edition** (or SonarQube Cloud) gives the same workflow with official support. Everything in steps 4–15 stays the same.

---

*If this helped you, or you've found a better way to do any of these steps, I'd like to hear about it.*
