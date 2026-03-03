# Copilot Instructions for texlive-full

## Repository Summary
This repository builds and publishes multi-version **TeXLive Docker images** for use with Overleaf Server Pro and Overleaf-CEP sandboxed compile. There is no application code — all changes are to Dockerfiles and supporting configuration files. The images are published to GitHub Container Registry (`ghcr.io/ayaka-notes/texlive-full`) and Aliyun.

## Repository Layout
```
.
├── Makefile                    # Stub only (echo statements, no real targets)
├── docker-compose.yaml         # Build/push helper for all versions
├── .gitmodules                 # Submodule: texlive/Base/extrafonts -> overleaf-fonts
├── .github/
│   └── workflows/
│       ├── texlive(gchr).yaml  # CI: build+push versioned images to ghcr.io on push/dispatch
│       ├── texlive(aliyun).yaml# CI: build+push versioned images to Aliyun
│       ├── base.yaml           # CI: build+push base and slim images (manual dispatch only)
│       └── clean.yaml          # Scheduled: delete untagged containers daily
└── texlive/
    ├── Base/
    │   ├── Dockerfile          # Base image (Ubuntu 22.04 + fonts + R + utilities)
    │   └── extrafonts/         # Git submodule: overleaf-fonts (OTF/TTF extra fonts)
    ├── Slim/
    │   └── Dockerfile
    ├── Test/
    │   └── Dockerfile          # Minimal (ubuntu:latest) for push-testing only
    ├── 2020/ … 2026/           # One directory per TeXLive release year
    │   ├── Dockerfile          # FROM base; installs TeXLive from Utah archive mirror
    │   ├── LatexMk             # latexmk config (Perl): knitr, glossaries, chktex, feynmf…
    │   ├── patchSynctex.R      # R script: patches .synctex files for knitr concordance
    │   ├── run-chktex.sh       # Bash wrapper: runs chktex with ulimit
    │   └── 09-texlive.conf     # fontconfig XML: exposes TeXLive fonts to XeTeX/LuaTeX
```

### Version image pattern
Every versioned Dockerfile (`texlive/20XX/Dockerfile`) follows the same pattern:
1. `FROM ghcr.io/ayaka-notes/texlive-full:base` — must exist before building any version image.
2. Copies `LatexMk`, `patchSynctex.R`, `run-chktex.sh`, `09-texlive.conf`.
3. Downloads and installs TeXLive from `https://ftp.math.utah.edu/pub/tex/historic/systems/texlive/<YEAR>/tlnet-final`.
4. Configures `texmf.cnf` (shell-escape, openout/openin, OSFONTDIR).
5. Sets `PATH` for the installed TeXLive binaries.
- TeXLive 2024 adds an extra `pip3 install latexminted==0.5.1` step.
- `09-texlive.conf` contains the literal year in its font directory paths and must be updated per version.

## Building

### Prerequisites
- Docker ≥ 20 with BuildKit.
- The **base image** (`ghcr.io/ayaka-notes/texlive-full:base`) must be available (pulled or built locally) before building any versioned image, because all versioned Dockerfiles use it as their `FROM`.
- Initialize the git submodule before building the base image:
  ```bash
  git submodule update --init --recursive
  ```

### Build base image
```bash
docker build -t ghcr.io/ayaka-notes/texlive-full:base ./texlive/Base
```
Building the base image requires internet access to download Google Fonts and Ubuntu packages. It takes 15–30 minutes.

### Build a specific versioned image
```bash
# Replace 2024 with the desired year
docker build -t ghcr.io/ayaka-notes/texlive-full:2024.1 ./texlive/2024
```
Building a versioned image requires the base image and internet access to the Utah TeXLive archive. Full TeXLive installation takes **60–120+ minutes**.

### Build via docker-compose
```bash
REGISTRY_AND_USER=ghcr.io/ayaka-notes docker-compose build 2024
REGISTRY_AND_USER=ghcr.io/ayaka-notes docker-compose push 2024
```

## CI / GitHub Actions Workflows

| Workflow | Trigger | What it does |
|---|---|---|
| `texlive(gchr).yaml` | Push to `main` touching `texlive/20XX/**`; or `workflow_dispatch` | Builds & pushes changed (or all) versioned images to ghcr.io |
| `texlive(aliyun).yaml` | Push to `main` touching `texlive/20XX/**`; or `workflow_dispatch` | Same but to Aliyun registry |
| `base.yaml` | `workflow_dispatch` only | Builds & pushes base and slim images to ghcr.io |
| `clean.yaml` | Daily cron + `workflow_dispatch` | Deletes untagged containers from ghcr.io |

**Versioned build matrix** (`texlive(gchr).yaml`): On push to main, CI diffs changed paths under `texlive/` and builds only the affected year directories. If no year directories changed, it builds all: `["2020","2021","2022","2023","2024","2025","2026"]`.

**There are no unit tests or lint steps.** CI validation = successful Docker build + push.

## Adding a New TeXLive Version (e.g. 2027)
1. Copy an existing version directory: `cp -r texlive/2026 texlive/2027`.
2. In `texlive/2027/Dockerfile`: update `ARG TEXLIVE_Version=2027` and verify the Utah mirror URL exists for that year.
3. In `texlive/2027/09-texlive.conf`: replace the year number in all font directory paths.
4. Add a new service to `docker-compose.yaml`:
   ```yaml
   2027:
     image: ${REGISTRY_AND_USER}/texlive-full:2027.1
     build:
       context: ./texlive/2027
       dockerfile: Dockerfile
   ```
5. Add `'texlive/2027/**'` to the `paths` trigger in `.github/workflows/texlive(gchr).yaml` and `.github/workflows/texlive(aliyun).yaml`, and add `"2027"` to both `ALL` version arrays in those files.

## Key Dependencies and Notes
- **No linter** is configured; `Makefile` targets are stubs.
- Tags use `.1` suffix (e.g. `2024.1`) because Overleaf uses the tag number to identify the TeXLive version.
- The `tex` user (uid/gid 1000) is created in the base image to match Overleaf's compile container expectations.
- `TEXMFVAR=/tmp/texmf-var` is set so LuaTeX cache is always writable.
- `REBUILT_AFTER` env var in the base Dockerfile forces Docker layer cache invalidation when bumped.
- For China mainland users, `ghcr.io` can be replaced with `ghcr.nju.edu.cn`.
- Required secret: `ORGTOKEN` (org-scoped PAT) for submodule checkout in `base.yaml`; `ALIYUN_REGISTRY_TOKEN` for Aliyun pushes.

## Trust These Instructions
Trust the instructions above. Only search the codebase if information here is incomplete or appears incorrect.
