---
description: Resolve Python dependencies through the internal Artifactory proxy with 7-day age-gate awareness
disable-model-invocation: false
---

# PyPI Dependency Resolution (pypi-resolve)

## Problem

Black Duck's internal Artifactory proxy (`artifactory.tools.duckutil.net`) has a **7-day age gate**: packages uploaded to PyPI within the last 7 days are not yet mirrored. Installing unreplicated packages directly via `pip install` will fail with a 404 from the proxy.

## Solution: `pypi-resolve`

A `uv run --script` tool installed at `~/.local/bin/pypi-resolve` that:

1. Resolves the full transitive dependency tree from PyPI
2. Enforces the 7-day age gate (skips versions too new for Artifactory)
3. Optionally checks which packages exist in Artifactory vs. missing
4. Outputs a sorted `package==version` lock file with dependencies before dependents

## Usage

### Lock dependencies for a project (with age gate + Artifactory check)

```bash
pypi-resolve -a -g $(cat requirements.txt) -o requirements.lock
```

This outputs only packages **not yet in Artifactory** — making the diff minimal for manual review.

### Just check what's missing

```bash
pypi-resolve -a -g prophet numpy
```

### Block specific versions (e.g., known-vulnerable)

```bash
pypi-resolve -a -g -b jupyter-builder==1.0.2 -b numpy <other-packages>
```

Or from a file:
```bash
pypi-resolve -g -a --block-file ~/src/blocked.txt $(cat requirements.txt)
```

### Resolve without Artifactory (age gate only)

```bash
pypi-resolve -g prophet
```

### Flags

| Flag | Purpose |
|------|---------|
| `-a` | Filter out packages already present in Artifactory (requires `$ARTIFACTORY_TOKEN`) |
| `-g` | Enforce the 7-day age gate (step down to older safe versions) |
| `-b PKG==VER` | Block a specific version (repeatable) |
| `--block-file` | File of blocked entries (one per line, `#` comments) |
| `-o FILE` | Write output to file instead of stdout |
| `-v` | Show resolution progress on stderr |

## When to use

- **Setting up a new Python project** — resolve deps before the first `pip install`
- **Adding a new dependency** — check it's available in Artifactory before committing
- **Upgrading deps** — `pypi-resolve -g` finds the newest version that passes the age gate

## Prerequisites

`$ARTIFACTORY_TOKEN` must be set (or passed via `--artifactory-token`) when using `-a`.
