# GitHub Actions — CI/CD Deep Dive

**One line:** Automation platform built into GitHub — run tests, build, and deploy code automatically when events happen in your repo.

---

## Why It Matters

GitHub Actions is in job listings across every backend, DevOps, and full-stack role. It's the default CI/CD tool for open-source and startup projects. Understanding it means you can:
- Automate tests so bugs get caught before merge
- Build and ship Docker images automatically
- Deploy apps without manual steps
- Prove you understand modern software delivery in interviews

Pairs directly with skill #5 (Docker/Kubernetes) on the roadmap.

---

## Core Concepts

```
Repo event (push, PR, schedule)
    │
    ▼
Workflow  (.github/workflows/name.yml)
    │
    ├── Job 1 ──► runs-on: ubuntu-latest
    │       ├── Step 1: uses: actions/checkout@v4
    │       ├── Step 2: run: npm install
    │       └── Step 3: run: npm test
    │
    └── Job 2 ──► runs-on: ubuntu-latest
            ├── needs: [job1]        ← only runs if job1 passes
            └── Step 1: run: deploy
```

| Concept | What it is |
|---------|-----------|
| **Workflow** | YAML file in `.github/workflows/`. Defines automation. |
| **Event** | What triggers the workflow — push, PR, cron, manual |
| **Job** | Group of steps running on one machine |
| **Step** | Single task — a shell command or a pre-built action |
| **Action** | Reusable unit of code (like a plugin). `uses:` syntax |
| **Runner** | The VM that executes a job. GitHub-hosted or self-hosted |

---

## Workflow File Structure

```yaml
name: Workflow name (shown in GitHub UI)

on:                         # trigger(s)
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:                        # workflow-level env vars
  NODE_ENV: production

jobs:
  job-name:
    runs-on: ubuntu-latest  # runner OS
    env:                    # job-level env vars
      BUILD_DIR: ./dist
    steps:
      - name: Step label
        uses: owner/action@version   # pre-built action
        with:
          input: value

      - name: Shell command
        run: echo "hello"
        env:                # step-level env vars
          SECRET: ${{ secrets.MY_SECRET }}
```

---

## Events (Triggers)

```yaml
on: push                          # any push to any branch

on:
  push:
    branches: [main, dev]         # push to specific branches
    paths: ['src/**', '*.go']     # only if these files changed

on:
  pull_request:
    branches: [main]              # PR targeting main
    types: [opened, synchronize]  # PR events to react to

on:
  schedule:
    - cron: '0 9 * * 1'          # every Monday at 9am UTC

on:
  workflow_dispatch:              # manual trigger from GitHub UI
    inputs:
      environment:
        description: 'Deploy target'
        required: true
        default: 'staging'

on: [push, pull_request]         # multiple triggers shorthand
```

---

## 1. Running Tests Automatically

Trigger on every push and PR — catch bugs before they merge.

```yaml
name: Run Tests

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - run: npm ci
      - run: npm test
```

**Language equivalents:**

| Language | Setup action | Test command |
|----------|-------------|-------------|
| Go | `actions/setup-go@v5` | `go test ./...` |
| Python | `actions/setup-python@v5` | `pytest` |
| Java | `actions/setup-java@v4` | `mvn test` |
| Rust | (built-in on runner) | `cargo test` |
| Node.js | `actions/setup-node@v4` | `npm test` |

---

## 2. Building and Deploying

### Sequence: test → build → deploy

```yaml
name: CI/CD

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm test

  build:
    runs-on: ubuntu-latest
    needs: test                   # only starts if test passes
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm run build
      - uses: actions/upload-artifact@v4
        with:
          name: dist
          path: ./dist

  deploy:
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/main'    # skip on PRs
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: dist
          path: ./dist
      - run: ./scripts/deploy.sh
```

### Deploy static site to GitHub Pages

```yaml
- uses: peaceiris/actions-gh-pages@v3
  with:
    github_token: ${{ secrets.GITHUB_TOKEN }}
    publish_dir: ./dist
```

### Build and push Docker image

```yaml
- uses: docker/login-action@v3
  with:
    username: ${{ secrets.DOCKER_USERNAME }}
    password: ${{ secrets.DOCKER_PASSWORD }}

- uses: docker/build-push-action@v5
  with:
    push: true
    tags: yourusername/appname:latest
```

---

## 3. Marketplace Actions

Actions are reusable steps published at [github.com/marketplace](https://github.com/marketplace?type=actions).

**Syntax:**
```yaml
- uses: owner/action-name@version
  with:
    input-param: value
```

**Always pin to a tag (`@v4`), never `@main`** — `@main` can change and silently break your workflow.

### Essential actions

| Action | Purpose |
|--------|---------|
| `actions/checkout@v4` | Clone repo onto runner — almost every workflow starts with this |
| `actions/setup-node@v4` | Install Node.js, optionally cache `node_modules` |
| `actions/setup-go@v5` | Install Go, optionally cache build cache |
| `actions/setup-python@v5` | Install Python |
| `actions/upload-artifact@v4` | Save files from a job (build output, test reports) |
| `actions/download-artifact@v4` | Retrieve files saved by upload-artifact in a later job |
| `actions/cache@v4` | Cache arbitrary directories to speed up runs |
| `docker/login-action@v3` | Log into Docker Hub or GHCR |
| `docker/build-push-action@v5` | Build and push Docker images |

### Using action outputs

```yaml
steps:
  - uses: actions/checkout@v4
    id: checkout

  - run: echo "Commit: ${{ steps.checkout.outputs.commit }}"
```

---

## 4. Secrets and Environment Variables

### Adding secrets

1. Repo → **Settings** → **Secrets and variables** → **Actions**
2. **New repository secret** → name + value → Save
3. Value is encrypted, never visible again after saving

```yaml
- run: ./deploy.sh
  env:
    API_KEY: ${{ secrets.MY_API_KEY }}

- uses: docker/login-action@v3
  with:
    password: ${{ secrets.DOCKER_PASSWORD }}
```

GitHub **masks secret values in logs** — they appear as `***` if printed.

### GITHUB_TOKEN

Auto-created by GitHub for every run. No setup needed. Scoped to your repo.

```yaml
github_token: ${{ secrets.GITHUB_TOKEN }}
```

### Env var scopes

```yaml
env:                          # workflow scope — all jobs and steps
  APP_NAME: myapp

jobs:
  build:
    env:                      # job scope — all steps in this job
      BUILD_DIR: ./dist
    steps:
      - run: npm test
        env:                  # step scope — only this step
          TEST_MODE: ci
```

### Built-in variables (no setup needed)

| Variable | Value |
|----------|-------|
| `GITHUB_REPOSITORY` | `owner/repo` |
| `GITHUB_REF` | `refs/heads/main` |
| `GITHUB_SHA` | Full commit SHA |
| `GITHUB_ACTOR` | Username who triggered the run |
| `GITHUB_WORKSPACE` | Path to checked-out code on runner |

### Variables (non-secret config)

For values that aren't sensitive — visible in logs.

Settings → **Secrets and variables** → **Variables tab** → add `APP_VERSION=2.1.0`

```yaml
run: echo "Building ${{ vars.APP_VERSION }}"
```

---

## 5. Multiple Jobs and Dependencies

### Parallel by default

All jobs start simultaneously unless `needs:` is specified.

```yaml
jobs:
  test:      # ─┐
    ...        #  ├── all start at the same time
  lint:      # ─┤
    ...        # │
  type-check:# ─┘
    ...
```

### Sequencing with `needs`

```yaml
jobs:
  test:
    ...
  build:
    needs: test           # waits for test
  deploy:
    needs: build          # waits for build
```

### Multiple dependencies

```yaml
deploy:
  needs: [test, lint]     # waits for BOTH
```

### Passing data between jobs

Jobs run on separate machines. Use artifacts for files, outputs for values.

**Artifacts (files):**
```yaml
jobs:
  build:
    steps:
      - run: npm run build
      - uses: actions/upload-artifact@v4
        with:
          name: dist-files
          path: ./dist

  deploy:
    needs: build
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: dist-files
          path: ./dist
```

**Outputs (strings/values):**
```yaml
jobs:
  get-version:
    outputs:
      version: ${{ steps.read.outputs.version }}
    steps:
      - id: read
        run: echo "version=$(cat version.txt)" >> $GITHUB_OUTPUT

  deploy:
    needs: get-version
    steps:
      - run: echo "Deploying ${{ needs.get-version.outputs.version }}"
```

### Matrix strategy

Run one job across multiple versions or OSes automatically.

```yaml
jobs:
  test:
    strategy:
      matrix:
        node-version: [18, 20, 22]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
      - run: npm test
```

Creates 3 parallel jobs. Multi-dimensional matrix:

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest, macos-latest]
    node: [18, 20]
# creates 6 jobs (every combination)
```

### Conditional execution

```yaml
jobs:
  deploy:
    if: github.ref == 'refs/heads/main'    # only on main branch

  notify:
    needs: deploy
    if: failure()                           # only if deploy failed

  always-run:
    if: always()                            # runs regardless of prior failures
```

### Failure control

```yaml
strategy:
  fail-fast: false        # don't cancel other matrix jobs if one fails
  matrix:
    node: [18, 20, 22]

jobs:
  optional:
    continue-on-error: true   # workflow passes even if this job fails
```

---

## Runners

| Runner | OS | Notes |
|--------|----|-------|
| `ubuntu-latest` | Ubuntu 24.04 | Fastest, cheapest, most common |
| `windows-latest` | Windows Server 2022 | Use for Windows-specific builds |
| `macos-latest` | macOS 14 | Required for iOS/macOS builds; slower |
| Self-hosted | Your machine | On-prem, custom hardware, no usage limits |

GitHub-hosted runners include: Docker, Node, Python, Go, Java, Git, and most common CLI tools pre-installed.

---

## Caching Dependencies

Without caching, every run re-downloads all dependencies. Cache cuts run time significantly.

```yaml
- uses: actions/setup-go@v5
  with:
    go-version: '1.22'
    cache: true               # setup-go handles Go module caching automatically

- uses: actions/setup-node@v4
  with:
    node-version: '20'
    cache: 'npm'              # caches node_modules automatically
```

Manual cache for anything else:
```yaml
- uses: actions/cache@v4
  with:
    path: ~/.cache/pip
    key: ${{ runner.os }}-pip-${{ hashFiles('requirements.txt') }}
    restore-keys: |
      ${{ runner.os }}-pip-
```

Cache is invalidated when the key changes (e.g., `requirements.txt` changes). `restore-keys` provides fallback partial matches.

---

## Complete Real-World Example

Go project — test, build binary, publish release on tag push.

```yaml
name: CI/CD

on:
  push:
    branches: [main]
    tags: ['v*']
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: '1.22'
          cache: true
      - run: go test ./...
      - run: go vet ./...

  build:
    runs-on: ubuntu-latest
    needs: test
    strategy:
      matrix:
        goos: [linux, windows, darwin]
        goarch: [amd64, arm64]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: '1.22'
      - run: GOOS=${{ matrix.goos }} GOARCH=${{ matrix.goarch }} go build -o bin/app-${{ matrix.goos }}-${{ matrix.goarch }}
      - uses: actions/upload-artifact@v4
        with:
          name: binary-${{ matrix.goos }}-${{ matrix.goarch }}
          path: bin/

  release:
    runs-on: ubuntu-latest
    needs: build
    if: startsWith(github.ref, 'refs/tags/v')   # only on version tags
    steps:
      - uses: actions/download-artifact@v4
        with:
          path: ./binaries
          pattern: binary-*
          merge-multiple: true
      - uses: softprops/action-gh-release@v2
        with:
          files: binaries/**
```

---

## Key Patterns to Remember

| Pattern | How |
|---------|-----|
| Run only on main | `if: github.ref == 'refs/heads/main'` |
| Run only on tags | `if: startsWith(github.ref, 'refs/tags/')` |
| Skip a job on PR | `if: github.event_name != 'pull_request'` |
| Pass file between jobs | upload-artifact + download-artifact |
| Pass value between jobs | `$GITHUB_OUTPUT` + job `outputs:` |
| Run on multiple versions | `strategy.matrix` |
| Only deploy after tests | `needs: test` on deploy job |
| Keep secrets out of logs | `${{ secrets.NAME }}` — GitHub masks automatically |
