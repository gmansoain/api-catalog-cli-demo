---
title: "Automating API Catalog Publishes with GitHub Actions"
slug: "automating-api-catalog-publishes-with-github-actions"
description: "Turn the manual API Catalog CLI publish from Part 1 into a hands-off GitHub Actions workflow that promotes RAML and OAS specs to Anypoint Exchange on every merge to main."
author: "Gonzalo Marcos"
date: 2026-09-01
lang: en
category: api-governance         # ⚠ REVIEW: same as Part 1 — align with the closest slug in _setup/bloggon-mulesoft-standards.md
tags:
  - api-catalog
  - anypoint-exchange
  - github-actions
  - ci-cd
  - devops
  - english
series: "API Catalog CLI"
series_part: 2
type: tutorial
difficulty: intermediate
read_time: 12
# ── domain-specific fields (mulesoft) ──
mule_version: n-a                # ⚠ REVIEW: control-plane tool — align with the module's "not applicable" value
platform: anypoint-exchange      # ⚠ REVIEW: align with the module's Platform taxonomy value
status: not validated
canonical_url: ""
---

![Not Validated](https://img.shields.io/badge/Status-Not_Validated-f1c40f) ![English](https://img.shields.io/badge/Lang-English-4a4a4a) ![Mule Version](https://img.shields.io/badge/Mule_Version-N%2FA-6a89cc) ![Platform](https://img.shields.io/badge/Platform-Anypoint_Exchange-00A0DF) ![Intermediate](https://img.shields.io/badge/Level-Intermediate-f39c12) ![Tutorial](https://img.shields.io/badge/Type-Tutorial-8e44ad) ![Series](https://img.shields.io/badge/Series-API_Catalog_CLI-16a085) ![Part 2](https://img.shields.io/badge/Part-2-16a085) ![12 min](https://img.shields.io/badge/Read_Time-12_min-lightgrey)

# Automating API Catalog Publishes with GitHub Actions

In [Part 1](./publishing-apis-to-anypoint-exchange-with-the-api-catalog-cli-on-ubuntu.md) we installed the API Catalog CLI on an Ubuntu box, wired it to Anypoint, and published three specs with a single command. That command works — but a human keeps having to run it, and humans forget. The whole point of a descriptor-driven publish is to let CI take over: any spec change lands in Exchange **without** anyone SSHing into a server.

In this post we'll do exactly that: turn the manual `api-catalog publish-asset` into a **GitHub Actions** workflow that runs on every push to `main`, plus a one-click **manual trigger** for when we want to force a re-publish. Same descriptor, same three sample APIs, same Connected App — just no more Ubuntu box.

We'll also answer two design questions that always come up around this: *where should the catalog live?* and *should we run this on a schedule?* Short answers: **a dedicated repo**, and **no, but here's what to do instead**.

> 🔗 **Previous:** [Part 1 — Publishing APIs to Anypoint Exchange with the API Catalog CLI on Ubuntu](./publishing-apis-to-anypoint-exchange-with-the-api-catalog-cli-on-ubuntu.md) · **Series:** API Catalog CLI

## Prerequisites

- Completed **Part 1**, or already have the four Connected App values ready: `client_id`, `client_secret`, host (usually `anypoint.mulesoft.com`), and Business Group `organizationId`
- A **GitHub account or organization** with permission to create a repo, add Secrets, and enable Actions
- The three sample specs from the lab's `APIs/` folder and the `catalog.yaml` we built in Part 1

<details>
<summary><strong>Files we'll add to the repo in this tutorial</strong></summary>

| File | Purpose |
| --- | --- |
| `APIs/…` | The three sample specs (unchanged from Part 1) |
| [`catalog.yaml`](./APIs/catalog.yaml) | The catalog descriptor — the source of truth for what gets published |
| `.github/workflows/publish-catalog.yml` | The new workflow, defined in Step 3 |

</details>

## Where should the catalog live?

Before we open GitHub, one decision: **which repo owns the descriptor and the specs?** Two patterns cover 90% of real-world setups:

| Pattern | Fits when… | Trade-off |
| --- | --- | --- |
| **A dedicated `api-catalog` repo** | An Integration/CoE team governs the API landscape and every spec passes through them. | Simple CI, one workflow file, one set of Secrets. Requires cross-team PRs to land new specs. |
| **Per-team repos + reusable workflow** | Each product team owns its own APIs end to end and publishes independently. | Scales with team count, but needs a shared reusable workflow (or callable action), plus per-BG Connected Apps for isolation. |

For this tutorial we'll use **the dedicated repo** — it maps cleanly to a single Actions workflow, mirrors the folder we already have on disk from Part 1, and is what most orgs start with. The layout looks like this:

```text
api-catalog/
├── APIs/
│   ├── gon-flights-api-1.0.3-raml/
│   ├── orders-system-api-1.0.0-raml/
│   └── twilio-system-api-1.0.1-fat-oas/
├── catalog.yaml
└── .github/
    └── workflows/
        └── publish-catalog.yml
```

> [!NOTE]
> If we later split into the per-team pattern, the workflow file becomes a **reusable workflow** (`workflow_call:`) that each team's repo invokes with its own descriptor path and Secrets. The publish logic itself doesn't change — only who owns the trigger.

## Step 1 — Push the lab to a GitHub repo

From the `API Catalog CLI` folder we've been using in Part 1, create the repo and push:

```bash
# From the lab folder — the one containing APIs/ and catalog.yaml
gh repo create api-catalog --public --source=. --remote=origin --push
```

Or, if we prefer the classic route:

```bash
git init
git add APIs/ catalog.yaml
git commit -m "chore: initial import — three sample specs + descriptor"
git branch -M main
git remote add origin git@github.com:<YOUR_ORG>/api-catalog.git
git push -u origin main
```

> [!WARNING]
> The lab's `api-catalog-cli-snippets.txt` and any local config files can contain real credentials. Add them to `.gitignore` **before** the first push. A leaked Connected App secret means anyone can publish to your Exchange until you rotate it.

> 📸 *Screenshot: `gh repo create` output showing the new repo URL.*

### Turn on branch protection for `main`

In **Settings → Branches → Add rule**, protect `main` with at minimum:

- **Require a pull request before merging** — prevents direct pushes bypassing review
- **Require status checks to pass before merging** — will start applying once the workflow has run at least once

The publish workflow runs *after* merge, so protection here isn't about blocking bad code from CI — it's about making sure a human approves every change that will land in Exchange.

## Step 2 — Store the Anypoint credentials as GitHub Secrets

The workflow needs the same four values we typed into `api-catalog conf` in Part 1. Store them under **Settings → Secrets and variables → Actions → New repository secret**:

| Secret name | Value |
| --- | --- |
| `ANYPOINT_CLIENT_ID` | Connected App client ID |
| `ANYPOINT_CLIENT_SECRET` | Connected App client secret |
| `ANYPOINT_HOST` | `anypoint.mulesoft.com` (or `eu1.anypoint.mulesoft.com`, `gov.anypoint.mulesoft.com`) |
| `ANYPOINT_ORG_ID` | Business Group organization ID |

> [!IMPORTANT]
> Use a **dedicated Connected App for CI** — do not reuse a developer's personal app. Scope it to the target Business Group with `Exchange Contributor` (and `Exchange Administrator` if we want to update existing assets). Rotate the secret on a schedule the security team is comfortable with; a rotation is a one-liner in this workflow.

> 📸 *Screenshot: the Secrets page with the four `ANYPOINT_*` entries.*

## Step 3 — Author the workflow

Create `.github/workflows/publish-catalog.yml` with the following content. We'll unpack the interesting pieces immediately after.

```yaml
# .github/workflows/publish-catalog.yml
name: Publish API Catalog

on:
  push:
    branches: [main]
    paths:
      - "APIs/**"
      - "catalog.yaml"
      - ".github/workflows/publish-catalog.yml"
  workflow_dispatch: {}

concurrency:
  group: publish-catalog-${{ github.ref }}
  cancel-in-progress: false

permissions:
  contents: read

jobs:
  publish:
    name: Publish to Anypoint Exchange
    runs-on: ubuntu-latest
    environment: anypoint-production   # optional — see Hardening below

    steps:
      - name: Check out the repo
        uses: actions/checkout@v4

      - name: Set up Node.js 20
        uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: npm

      - name: Install API Catalog CLI
        run: npm install -g api-catalog-cli@latest

      - name: Configure CLI credentials
        env:
          ANYPOINT_CLIENT_ID:     ${{ secrets.ANYPOINT_CLIENT_ID }}
          ANYPOINT_CLIENT_SECRET: ${{ secrets.ANYPOINT_CLIENT_SECRET }}
          ANYPOINT_HOST:          ${{ secrets.ANYPOINT_HOST }}
          ANYPOINT_ORG_ID:        ${{ secrets.ANYPOINT_ORG_ID }}
        run: |
          api-catalog conf client_id     "$ANYPOINT_CLIENT_ID"
          api-catalog conf client_secret "$ANYPOINT_CLIENT_SECRET"
          api-catalog conf host          "$ANYPOINT_HOST"
          api-catalog conf organization  "$ANYPOINT_ORG_ID"

      - name: Publish descriptor
        run: api-catalog publish-asset -d catalog.yaml
```

📄 Full file: [.github/workflows/publish-catalog.yml](./.github/workflows/publish-catalog.yml)

### Why each block is there

- **`on.push` with `paths:`** — the workflow only fires when specs, the descriptor, or the workflow itself change. A README typo won't burn a CI minute.
- **`on: workflow_dispatch: {}`** — adds the **"Run workflow"** button in the Actions tab. Useful after rotating the Connected App secret, after fixing a taxonomy in Exchange, or when we simply want to prove the pipeline still works.
- **`concurrency`** — if two merges land in quick succession, GitHub queues the second run until the first finishes. `cancel-in-progress: false` because a publish that's mid-upload should not be cancelled and left in an unknown state.
- **`permissions: contents: read`** — least-privilege. The workflow only reads the checkout; it doesn't push tags or comment on PRs.
- **`environment: anypoint-production`** — declaring an **Environment** lets us gate the job behind a manual approval, restrict which branches can trigger it, and scope the Secrets to that environment. Optional but recommended once the workflow is live.
- **`actions/setup-node@v4` with `cache: npm`** — reuses the previous run's npm cache so the CLI install stays snappy across runs.
- **Credentials via `env:`** — passing Secrets through `env:` instead of interpolating them directly in `run:` keeps them out of the shell history and out of the job log even if `set -x` is enabled.

> [!TIP]
> The `api-catalog` config file is written to the runner's `$HOME` and vanishes with the runner. Every job starts clean — that's why we re-run `conf` on every publish.

## Step 4 — Commit, push, and watch it run

```bash
git add .github/workflows/publish-catalog.yml
git commit -m "ci: publish api catalog on push to main"
git push
```

Open the repo's **Actions** tab. The **Publish API Catalog** workflow should be running. When it finishes, the last step logs something like:

```diff
+ [gon-flights-api]      packaged (RAML)      → uploaded  1.0.3
+ [orders-system-api]    packaged (RAML)      → uploaded  1.0.0
+ [twilio-system-api]    packaged (OAS)       → uploaded  1.0.1
Done. 3/3 assets published.
```

> 📸 *Screenshot: green check on the Actions run and the three assets appearing in Anypoint Exchange.*

Head to Exchange, filter by the `demo` tag, and confirm the three assets are there — same as at the end of Part 1, but now landed there without a single terminal command.

## Step 5 — Fire the manual trigger

Back in the Actions tab, pick **Publish API Catalog** in the left sidebar and click **Run workflow → main → Run workflow**. This uses the `workflow_dispatch` event we declared and runs the exact same job.

> 📸 *Screenshot: the "Run workflow" dropdown on the Actions page.*

Two situations where this button earns its keep:

- **Right after rotating the Connected App secret** — no code changed, but we want to prove the new credential works before the next real merge relies on it.
- **After fixing an Exchange category** — an earlier publish silently dropped a `categories:` value because the key didn't exist yet in Exchange; we've since added it and want to re-publish so the metadata lands.

## What re-runs actually do

This is where the "should we schedule it?" question lands. Exchange asset versions are **immutable** — once `gon-flights-api 1.0.3` exists, it can't be overwritten. So re-running `publish-asset` on the same descriptor behaves like this:

| Situation | What happens in Exchange | What the job does |
| --- | --- | --- |
| Descriptor unchanged, same `version:` | Nothing — the version is already there | CLI returns `409 Conflict — version already exists` for each asset; job **fails** |
| `version:` bumped (e.g. `1.0.3` → `1.0.4`) | The new version is created alongside the old one | Job **succeeds**; consumers can switch versions from the dropdown |
| `assetId:` renamed | A brand-new asset is created; the old one stays orphaned | Job **succeeds**, but Exchange now has a ghost — clean it up manually |
| `categories:` or `tags:` changed, `version:` unchanged | Metadata does **not** update on an existing version | Job either 409s or no-ops depending on CLI version — bump `version:` to force a refresh |

That last row is the subtle one: **metadata is bound to a version**. If we want a category fix to show up, we bump the asset version.

> [!TIP]
> This is exactly why **we don't put the publish on a `schedule:` cron**. A nightly run against an unchanged descriptor either fails loudly (409) or wastes minutes doing nothing. If a scheduled sweep still sounds valuable, the right shape is a **drift-check job** — list every asset in Exchange, diff against `catalog.yaml`, and open an issue if they diverge (someone published manually via the UI, or an asset was deleted). That's a separate workflow calling Exchange's REST API — described here, not implemented, because it deserves its own post.

## Hardening

A short checklist for taking this from "it works" to "it belongs in production":

- **Gate publishes behind an approval.** Under **Settings → Environments → anypoint-production**, add a required reviewer. The job will pause after `checkout` and wait for a human before touching Exchange.
- **Restrict which branches can trigger the environment.** In the same page, allow only `main`. This blocks a rogue feature branch from claiming the credentials.
- **Split by Business Group.** For orgs with multiple BGs, turn `publish` into a `strategy.matrix` over `{ org_id, secret_prefix }` pairs — each matrix job publishes to its own BG with its own Connected App.
- **Log-scrub the descriptor.** The default log doesn't print secrets, but adding `::add-mask::` for the org ID keeps it out of screenshots people paste into support tickets.
- **Pin action versions.** `actions/checkout@v4` is fine to start; for higher-trust environments, pin to a full SHA (`actions/checkout@<sha>`) so a compromised tag can't swap the action.
- **Federated auth (OIDC).** GitHub → Anypoint OIDC federation would remove long-lived secrets entirely, but Anypoint's Connected Apps currently authenticate via **client_credentials only** — no OIDC path yet.  <!-- ⚠ REVIEW: confirm no OIDC federation option exists at time of writing -->

## Troubleshooting

| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| `401 Unauthorized` on `publish-asset` | Wrong secret value, or the Connected App was deleted / secret rotated | Re-check the four `ANYPOINT_*` secrets in **Settings → Secrets**; regenerate the Connected App secret and update the value |
| `403 Forbidden` for a specific asset | The Connected App has no permission on the target Business Group | Grant `Exchange Contributor` on the BG whose ID we stored in `ANYPOINT_ORG_ID` |
| `409 Conflict — version already exists` | Trying to re-publish an unchanged `version:` | Bump `version:` in `catalog.yaml` (or accept that this is a no-op and no publish is needed) |
| Workflow doesn't fire on push | The changed files don't match the `paths:` filter | Confirm the diff touches `APIs/**`, `catalog.yaml`, or the workflow file itself — or use **Run workflow** to trigger it manually |
| `command not found: api-catalog` | Node version too old, or install step skipped by cache | Ensure `actions/setup-node@v4` is set to `node-version: "20"` and that the `Install API Catalog CLI` step ran (not just cached) |
| Category dropped silently | Category key doesn't exist in Exchange yet | Add the key under **Exchange → Customize → Categories**, bump `version:` in the descriptor, re-run |

## Summary

We took Part 1's manual publish and moved it into a repo, driven by two triggers: a `push` to `main` (paths-filtered so it only fires on real spec changes) and a `workflow_dispatch` button for on-demand runs. Along the way we picked the **dedicated `api-catalog` repo** as the primary layout, gave the workflow least-privilege permissions, and explained why a `schedule:` cron is **not** the right tool for a publish that talks to an immutable version store.

> ➡️ **Next up:** In the next post we'll expand this pipeline with **PR-time validation** — a companion job that runs `create-descriptor` and a spec-lint step so contributors see errors *before* their change lands on `main`, and only clean descriptors ever reach Exchange.  <!-- ⚠ REVIEW: teaser assumes a Part 3; drop or reshape if this is the end of the series -->

## References

- [API Catalog CLI — official documentation](https://docs.mulesoft.com/exchange/api-catalog-cli "MuleSoft docs: API Catalog CLI")
- [GitHub Actions — `on.push.paths`](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#onpushpull_requestpull_request_targetpathspaths-ignore "GitHub docs: paths filter")
- [GitHub Actions — `workflow_dispatch`](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#workflow_dispatch "GitHub docs: manual trigger")
- [`actions/setup-node`](https://github.com/actions/setup-node "actions/setup-node on GitHub")
- [Encrypted secrets for a repository](https://docs.github.com/en/actions/security-guides/using-secrets-in-github-actions "GitHub docs: repository Secrets")
- [Using environments for deployment](https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment "GitHub docs: environments and approvals")
- [Anypoint Connected Apps — client credentials flow](https://docs.mulesoft.com/access-management/connected-apps-overview "MuleSoft docs: Connected Apps")
