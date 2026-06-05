# GitHub Pipeline - the real CI/CD running on GitHub Actions

This guide describes the actual pipeline that runs on GitHub
(`.github/workflows/cicd.yml`): what triggers it, what each job does, and which
actions are manual versus automated.

Each step is marked **[MANUAL]** (performed by the user) or **[AUTO]** (performed
automatically by GitHub Actions).

> Summary: the user performs `git push`; GitHub performs lint, test, build and deploy.

---

## Overview

```
   [MANUAL] git push
        |
        v
+-----------------------------------------------------------+
|              GitHub Actions  ([AUTO] fully automatic)     |
|                                                           |
|   [lint]  ->  [test]  ->  [build-and-push]                |
|   ruff        pytest       docker build + push to GHCR    |
+-----------------------------------------------------------+
        |
        v
   Image published to ghcr.io
```

## Triggers (`on:`)

```yaml
on:
  push:
    branches: ["main", "master"]
  pull_request:
    branches: ["main", "master"]
```

| Event | Jobs that run | Publishes image? |
|-------|---------------|------------------|
| [MANUAL] Push to `main`/`master` | lint -> test -> build-and-push | Yes |
| [MANUAL] Open / update a PR | lint -> test | No |

> The publish job has `if: github.event_name == 'push'`, so a PR validates the code
> but does not publish an image. This avoids publishing from unmerged branches.

---

## One-time setup [MANUAL]

### Setup 1 - Create the repo and push

```bash
git add .                                              # [MANUAL]
git commit -m "Initial commit: CI/CD pipeline demo"    # [MANUAL]
gh repo create cicd-pipeline-demo --public --source=. --push   # [MANUAL]
```

Alternatively, create an empty repo on github.com, then:

```bash
git remote add origin https://github.com/<user>/cicd-pipeline-demo.git   # [MANUAL]
git push -u origin master                              # [MANUAL]
```

> The moment this push lands, the pipeline starts automatically.

### Setup 2 - Grant write permission (prevents the 403 error)

Without this, the `build-and-push` job fails with 403 when publishing to GHCR.

In the repo on GitHub:
**Settings -> Actions -> General -> Workflow permissions ->**
select **"Read and write permissions"** -> **Save**. [MANUAL]

If the code was pushed before changing this, re-run the pipeline:

```bash
gh run rerun <run-id>     # [MANUAL]  (or push an empty commit)
```

---

## Job 1 - `lint` [AUTO]

```yaml
- uses: actions/checkout@v6        # [AUTO] checks out the code
- uses: actions/setup-python@v6    # [AUTO] installs Python 3.12
- run: pip install ruff==0.15.16   # [AUTO] installs the linter
- run: ruff check app/             # [AUTO] analyzes the code
```

**Purpose:** static/style analysis of `app/`.
**If it fails:** the pipeline stops here - `test` and `build-and-push` do not run.
**Manual involvement:** only if it fails - fix the code and push again.

---

## Job 2 - `test` [AUTO]

```yaml
needs: lint                        # [AUTO] only runs if lint passed
- uses: actions/checkout@v6
- uses: actions/setup-python@v6
- run: pip install -r requirements.txt
- run: pytest test_main.py -v      # [AUTO] runs the tests
```

**Purpose:** installs dependencies and runs the test suite (`app/test_main.py`).
**`needs: lint`:** waits for `lint` to succeed first.
**If it fails:** the pipeline stops; no image is built or published.
**Manual involvement:** only if it fails - fix the test or code and push again.

---

## Job 3 - `build-and-push` [AUTO]

```yaml
needs: test                        # [AUTO] only runs if test passed
if: github.event_name == 'push'    # [AUTO] only on push, not on PR
permissions:
  contents: read
  packages: write                  # [AUTO] permission to publish to GHCR
```

Internal steps (all [AUTO]):

1. **Log in to GHCR** - uses the automatic `GITHUB_TOKEN`; no secrets to create.
   ```yaml
   - uses: docker/login-action@v4
     with:
       registry: ghcr.io
       username: ${{ github.actor }}
       password: ${{ secrets.GITHUB_TOKEN }}
   ```

2. **Generate tags** - computes the image names automatically.
   ```yaml
   - uses: docker/metadata-action@v6
     with:
       images: ghcr.io/${{ github.repository }}
       tags: |
         type=sha,prefix=sha-     # e.g. sha-a1b2c3d
         type=raw,value=latest    # latest
   ```

3. **Build + Push** - builds from the `Dockerfile` and pushes.
   ```yaml
   - uses: docker/build-push-action@v7
     with:
       context: .
       push: true
   ```

**Purpose:** builds the Docker image and publishes it to
`ghcr.io/<user>/cicd-pipeline-demo` with two tags: `latest` and `sha-<commit>`.
**Manual involvement:** none during execution. Only the one-time setup above.

---

## Watch the run [AUTO] (observation only)

```bash
gh run watch        # [MANUAL] open the live viewer / [AUTO] shows progress in real time
gh run list         # [MANUAL] see run history
gh run view --log   # [MANUAL] full logs if something fails
```

Alternatively, use the repo's **Actions** tab in the browser.

Expected sequence ([AUTO]):

```
[AUTO] lint            passes
[AUTO] test            passes
[AUTO] build-and-push  image published to ghcr.io
```

---

## Verify the published image [MANUAL]

1. On GitHub: profile -> **Packages** -> `cicd-pipeline-demo` should appear. [MANUAL]
2. The package is private the first time. To make it public:
   **Package -> Package settings -> Change visibility -> Public**. [MANUAL]
3. Pull and run it:
   ```bash
   docker pull ghcr.io/<user>/cicd-pipeline-demo:latest             # [MANUAL]
   docker run -p 8000:8000 ghcr.io/<user>/cicd-pipeline-demo:latest # [MANUAL]
   ```

---

## Day-to-day workflow

```bash
# 1. [MANUAL] make code changes
# 2. [MANUAL] (recommended) validate locally - see 01-local-testing.md
ruff check app/ && (cd app && pytest)

# 3. [MANUAL] commit and push
git add .
git commit -m "describe the change"
git push

# 4. [AUTO] the pipeline does the rest:
#    lint -> test -> build -> push the new image
```

### Pull Request path (validates without publishing)

```bash
git checkout -b my-branch     # [MANUAL]
# ...changes...
git push -u origin my-branch  # [MANUAL]
gh pr create                  # [MANUAL] open the PR
# [AUTO] runs lint + test, but does not publish an image
```

---

## Responsibility summary (GitHub)

| Stage | Action | Performed by | Frequency |
|-------|--------|--------------|-----------|
| Setup | Create repo + first push | [MANUAL] | Once |
| Setup | "Read and write permissions" | [MANUAL] | Once |
| Trigger | `git push` | [MANUAL] | Every change |
| lint | ruff checks the code | [AUTO] | Every push/PR |
| test | pytest runs tests | [AUTO] | Every push/PR |
| build | docker build | [AUTO] | Every push |
| login | auth with GITHUB_TOKEN | [AUTO] | Every push |
| publish | docker push to GHCR | [AUTO] | Every push |
| Verify | check / make package public | [MANUAL] | Once (visibility) |

> Core rule: the user performs `git push`; the pipeline performs lint, test, build
> and deploy.
