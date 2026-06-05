# Local Testing - run the pipeline on the machine

This guide reproduces, on a local machine, every step the CI/CD pipeline runs on
GitHub. The purpose is to catch problems before pushing, so the real pipeline stays
green.

Each step is marked **[MANUAL]** (performed by the user) or **[AUTO]** (performed
automatically by the tool).

> The GitHub pipeline runs the exact same commands. If lint, tests and the Docker
> build pass here, they will most likely pass on GitHub.

---

## What is replicated

The GitHub workflow (`.github/workflows/cicd.yml`) runs three jobs in order:

```
[lint]  ->  [test]  ->  [build-and-push]
 ruff       pytest       docker build + push
```

Locally the first two are run exactly, and the build step is run without the push.


### Step 0 - Prerequisites [MANUAL]

The following must be available:

```bash
python --version    # [MANUAL] Python 3.12+
docker --version    # [MANUAL] Docker (for the build/run steps)
```

Move into the project folder:

```bash
cd "cicd-pipeline-demo"     # [MANUAL]
```


### Step 1 - Install dependencies [MANUAL]

Mirrors the pipeline's `pip install -r requirements.txt`.

```bash
pip install -r requirements.txt     # [MANUAL]
```


### Step 2 - Lint (same as the `lint` job) [MANUAL]

```bash
ruff check app/     # [MANUAL]
```

| Result | Meaning | Action |
|--------|---------|--------|
| `All checks passed!` | Code style is clean | Continue to Step 3 |
| Errors listed | Style/static issues | [MANUAL] Fix them and re-run. `ruff check app/ --fix` applies auto-fixes |

> On GitHub, if this fails the whole pipeline stops here: `test` and
> `build-and-push` never run. This is why it is checked first.


### Step 3 - Tests (same as the `test` job) [MANUAL]

```bash
cd app
pytest test_main.py -v     # [MANUAL]
cd ..
```

Expected output: 3 tests passing (`test_root`, `test_health`, `test_get_item`).

| Result | Action |
|--------|--------|
| `3 passed` | Continue to Step 4 |
| Failures | [MANUAL] Fix the code or the test and re-run |

### Step 4 - Build the Docker image (same as `build-and-push`, minus the push) [MANUAL]

```bash
docker build -t cicd-pipeline-demo:test .     # [MANUAL] starts the build; [AUTO] Docker reads the Dockerfile and builds
```

[AUTO] Docker installs dependencies inside the image, copies `app/`, and sets the
startup command (`uvicorn main:app`).


### Step 5 - Run and verify the image [MANUAL]

```bash
docker run -d -p 8000:8000 --name demo cicd-pipeline-demo:test   # [MANUAL] start container
curl http://localhost:8000/health                                # [MANUAL] -> {"status":"healthy"}
curl http://localhost:8000/                                      # [MANUAL] -> greeting + status
curl http://localhost:8000/items/42                              # [MANUAL] -> item detail
docker rm -f demo                                                # [MANUAL] stop & clean up
```

Alternatively, run everything with Docker Compose:

```bash
docker compose up --build     # [MANUAL] starts; [AUTO] builds + runs with healthcheck
# in another terminal:
curl http://localhost:8000/health     # [MANUAL]
docker compose down           # [MANUAL] stop
```

---

## To run the full local check in one line: [MANUAL]
This is an alternative to doing all the previous steps, or just a shorter way to run the test if we have ran it already before and we know we have Python and Docker on the machine.

```bash
pip install -r requirements.txt \
  && ruff check app/ \
  && (cd app && pytest test_main.py -v) \
  && docker build -t cicd-pipeline-demo:test .
```

If this finishes without errors, the project is ready to push. See
[[02-github-pipeline.md]].


### Responsibility summary (local)

| Step | Action | Performed by |
|------|--------|--------------|
| 0 | Check prerequisites | [MANUAL] |
| 1 | Install dependencies | [MANUAL] |
| 2 | Lint with ruff | [MANUAL] runs / [AUTO] ruff analyzes |
| 3 | Run pytest | [MANUAL] runs / [AUTO] pytest executes |
| 4 | Build Docker image | [MANUAL] runs / [AUTO] Docker builds |
| 5 | Run & verify container | [MANUAL] |

> Every step here is manual / on-demand - nothing runs automatically on the local
> machine. Automation begins once the code is pushed to GitHub (see the next file).
