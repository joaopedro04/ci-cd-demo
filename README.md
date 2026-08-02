# ci-cd-demo

A minimal static site — plain HTML and CSS, no framework, no real build step — used as an
excuse to wire up a **complete CI/CD pipeline** with GitHub Actions and Vercel.

The site isn't the point. These two files are:

| File | Role | What it does |
| --- | --- | --- |
| [`.github/workflows/ci.yml`](.github/workflows/ci.yml) | **CI** — Continuous Integration | Tells you whether the code is healthy: validates the HTML and looks for broken links. Publishes nothing. |
| [`.github/workflows/deploy.yml`](.github/workflows/deploy.yml) | **CD** — Continuous Delivery | Ships to Vercel: a preview on every PR, production on every merge to `main`. |

---

## The flow

```
   you open a PR
        │
        ▼
   ┌─────────┐   failed   ┌──────────────────────┐
   │   CI    │───────────▶│ nothing gets shipped │
   │ lint +  │            └──────────────────────┘
   │ build   │
   └────┬────┘
        │ passed
        ▼
   ┌──────────────────┐
   │ PREVIEW deploy   │──▶ URL commented on the PR
   └──────────────────┘

   merge to main
        │
        ▼
   CI ──passed──▶ PRODUCTION deploy
```

**CI** answers "is this correct?". **CD** answers "is this live yet?". The rule tying them
together: nothing ships without passing CI first.

---

## Why two files, and how one blocks the other

Splitting CI from CD is the standard move: one responsibility per file. The catch is that
`needs:` — the keyword that makes one job wait for another — **does not cross workflow
files**. Two independent workflows would run in parallel, and the deploy would go out even
with a failing lint.

The solution used here: `ci.yml` is a **reusable workflow**. It declares

```yaml
on:
  workflow_call:
```

and `deploy.yml` calls it as if it were one of its own jobs:

```yaml
jobs:
  ci:
    uses: ./.github/workflows/ci.yml
    secrets: inherit

  preview:
    needs: ci      # ← the gate
```

Note that `ci.yml` does **not** listen to `pull_request` or `push` on its own. If it did, CI
would run twice per PR: once by itself, and once called by the deploy.

> **An alternative we didn't use:** the `workflow_run` trigger, which fires one workflow when
> another finishes. It works, but it always runs in the context of the default branch, which
> makes PR preview deploys considerably more awkward. For this case, `workflow_call` is
> simpler and easier to read.

---

## Why the Vercel CLI instead of the automatic integration

Vercel can connect to your GitHub repository and deploy on its own, without a single line of
YAML. That's the easy path — and exactly why we avoid it here: the deploy becomes a black box
you can't read, version, or explain.

In this project the deploy runs through the Vercel CLI inside the workflow. That's why
[`vercel.json`](vercel.json) contains:

```json
"git": { "deploymentEnabled": { "main": false } }
```

This **turns off Vercel's automatic deploys for the `main` branch**. Without that line, every
push would trigger two deploys — the workflow's and Vercel's — and the pipeline would lose
its purpose.

The deploy is three commands, and the order matters:

```bash
vercel pull --environment=preview   # fetch the project's config and env vars
vercel build                        # build HERE, on the GitHub runner
vercel deploy --prebuilt            # upload the finished output
```

`--prebuilt` is the interesting bit: instead of shipping source code for Vercel to build on
their side, the build happens on the runner, where you can see every log line. Vercel only
receives the final artifact.

---

## Setting it up from scratch

### 1. Create the Vercel project

```bash
npm install --global vercel
vercel login
vercel link
```

`vercel link` creates a local `.vercel/project.json` (git-ignored) containing:

```json
{ "orgId": "team_xxx", "projectId": "prj_xxx" }
```

Keep both values handy.

### 2. Generate an access token

At **vercel.com → Settings → Tokens**, create a token scoped to your project.

### 3. Register the secrets on GitHub

Under **Settings → Secrets and variables → Actions → New repository secret**, add all three:

| Secret | Where to find it |
| --- | --- |
| `VERCEL_TOKEN` | the token from step 2 |
| `VERCEL_ORG_ID` | `orgId` in `.vercel/project.json` |
| `VERCEL_PROJECT_ID` | `projectId` in `.vercel/project.json` |

The workflows read them via `${{ secrets.NAME }}`. They never show up in the logs — GitHub
masks them automatically.

### 4. Publish the repository

```bash
git init && git add . && git commit -m "chore: initial ci/cd pipeline"
```

Then create the repository on GitHub and push `main`.

---

## Running the checks locally

The exact same commands CI runs — no lint logic lives inside the YAML, it lives in
`package.json`:

```bash
npm ci
npm run lint
```

To view the site:

```bash
npx serve public
```

The link checker (`lychee`) is a separate binary, optional locally:

```bash
npx --yes @lycheeverse/lychee --config lychee.toml --root-dir "$PWD/public" "public/**/*.html"
```

`--root-dir` is what lets it resolve root-relative links like `/styles.css` while the files
are still on disk. It has to be an absolute path.

---

## Exercises

Each one exists to make a different part of the pipeline react:

1. **Break a link.** Swap one of the footer `href`s for a URL that doesn't exist, open a PR,
   and watch the lint job fail — and no deploy happen. This is the core demonstration.
2. **Break the HTML.** Drop the `lang` attribute from `<html>`, or put an `<h3>` directly
   inside a `<ul>`, and watch `html-validate` complain.
3. **Open a real PR.** Change some hero copy and follow it through: CI green → preview
   deployed → URL commented on the PR. Merge it and watch production update.
4. **Add a new job.** A Prettier formatting step, for instance. Where does it belong — in
   `ci.yml` alongside the lint, or after it with `needs:`?
5. **Make CI required.** Under Settings → Branches, require the CI check before a merge to
   `main` is allowed. That's when the pipeline stops being a suggestion.

---

## Structure

```
.
├─ .github/workflows/
│  ├─ ci.yml            # lint + verification build (reusable)
│  └─ deploy.yml        # preview on PRs, production on main
├─ public/
│  ├─ index.html        # the landing page
│  └─ styles.css        # plain CSS, with dark mode
├─ .htmlvalidate.json   # HTML validator rules
├─ lychee.toml          # link checker rules
├─ vercel.json          # static output + automatic deploys disabled
└─ package.json         # lint scripts, shared between local and CI
```
