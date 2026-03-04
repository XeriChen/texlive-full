# TeXLive-Full Docker Image Development Guide

## Project Overview
This repository builds comprehensive TeXLive Docker images for Overleaf Server Pro/CEP. Images range from TeXLive 2020-2026, providing nearly complete LaTeX distributions with preinstalled fonts and utilities to minimize compilation failures.

## Architecture

### Three-Tier Image Structure
1. **Base Image** ([texlive/Base/Dockerfile](../texlive/Base/Dockerfile)): Ubuntu 22.04 foundation with system dependencies, fonts, R packages, and the `tex` user (UID 1000)
2. **Version Images** (`texlive/202X/`): Build on base, install specific TeXLive year with full scheme and custom configurations
3. **Slim Variant** ([texlive/Slim/Dockerfile](../texlive/Slim/Dockerfile)): Standalone image without the base layer, minimal dependencies

### Key Components Per Version
- `Dockerfile`: TeXLive installation, security configs (`shell_escape=t`, `openout_any=a`)
- `LatexMk`: Custom latexmk configuration with knitr synctex patching hooks
- `patchSynctex.R`: Post-processing script for R markdown concordance files
- `run-chktex.sh`: Wrapper for chktex with ulimit controls
- `09-texlive.conf`: Fontconfig to expose system fonts to XeTeX/LuaTeX
- `latexminted/.latexminted_config` (2025+): Minted package security config

## Build System

### Docker Compose
```bash
# Build specific versions
REGISTRY_AND_USER=ghcr.io/yourname docker-compose build 2026 2025 base

# Push to registry
REGISTRY_AND_USER=ghcr.io/yourname docker-compose push 2026 2025
```

**Critical**: The `.1` suffix (e.g., `2026.1`) is mandated by Overleaf's version detection system, not a monthly release indicator.

### GitHub Actions Workflow
- **Trigger Logic** ([.github/workflows/texlive(gchr).yaml](../.github/workflows/texlive(gchr).yaml)): Detects changed `texlive/202X/` paths and builds only affected versions
- **Manual Dispatch**: Accepts comma-separated versions (`2026,2025`) or `all`
- **Parallel Builds**: Max 8 concurrent version builds to optimize CI time
- **Base Image** ([.github/workflows/base.yaml](../.github/workflows/base.yaml)): Built separately via workflow_dispatch (infrequent updates)

## Development Conventions

### Version-Specific Customizations
When adding features to a specific year:
1. Apply to the latest version (2026) first
2. Backport to older versions if universally applicable
3. Test with Overleaf CEP integration (see [README.md](../README.md#-overleaf-cep-usage))

### Critical Configuration Patterns

#### TexMF Security Config
All version Dockerfiles must set:
```dockerfile
RUN echo "shell_escape = t" >> /usr/local/texlive/${TEXLIVE_Version}/texmf.cnf && \
    echo "openout_any = a" >> /usr/local/texlive/${TEXLIVE_Version}/texmf.cnf && \
    echo "openin_any = a" >> /usr/local/texlive/${TEXLIVE_Version}/texmf.cnf
```
This enables `--shell-escape` for minted/pythontex while sandboxing via Docker.

#### Font Cache Pre-Building
Base image runs `fc-cache -f` after font installation to prevent 10+ minute cache rebuilds during first compilation ([known issue #1](https://github.com/ayaka-notes/texlive-full/issues/1)).

#### Minted Permissions Fix
For 2025+, place `.latexminted_config` in `/home/tex/latexminted` with `enable_cwd_config: True` to resolve permission errors ([issue #131](https://github.com/yu-i-i/overleaf-cep/issues/131)).

### TeXLive Installation Pattern
All versions follow this sequence (see [texlive/2026/Dockerfile](../texlive/2026/Dockerfile)):
1. GPG signature verification of install-tl-unx.tar.gz
2. SHA512 checksum validation
3. Profile-based installation: `scheme-full`, no docs/sources, no autobackup
4. Post-install: `tlmgr install latexmk texcount synctex etoolbox xetex`
5. Font system configuration via `OSFONTDIR` and fontconfig

### R/Knitr Integration
The [LatexMk](../texlive/2026/LatexMk) file detects `-concordance.tex` files and invokes [patchSynctex.R](../texlive/2026/patchSynctex.R) to align line numbers between .Rnw and .tex for proper synctex support.

## Testing Changes

### Local Testing
```bash
# Build single version
docker build -t test:2026 ./texlive/2026

# Test compilation
docker run --rm -v $(pwd)/test-docs:/work test:2026 bash -c \
  "cd /work && latexmk -pdf document.tex"
```

### Integration Verification
Update Overleaf CEP's `config/variables.env`:
```
TEX_LIVE_DOCKER_IMAGE=your-registry/texlive-full:2026.1
```

## Known Gotchas

1. **Historic Mirror Constraints**: Pre-2019 TeXLive requires HTTP (not HTTPS) and SHA256 (not SHA512)
2. **Base Image Dependency**: Version images pull `FROM ghcr.io/XeriChen/texlive-full:base` — update base first for font/dependency changes
3. **User Context**: Compilations run as `tex:tex` (UID/GID 1000), ensure file permissions match
4. **TEXMFVAR Path**: Set to `/tmp/texmf-var` to allow LuaTeX cache writes in sandboxed environments

## File References

- Base dependencies: [texlive/Base/Dockerfile](../texlive/Base/Dockerfile)
- Version template: [texlive/2026/Dockerfile](../texlive/2026/Dockerfile)
- Build automation: [.github/workflows/texlive(gchr).yaml](../.github/workflows/texlive(gchr).yaml)
- Usage documentation: [README.md](../README.md)
