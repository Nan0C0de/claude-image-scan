# Docker Image Scan

A [Claude Code](https://claude.com/claude-code) plugin that builds, scans, and analyzes container images for security vulnerabilities using [Trivy](https://github.com/aquasecurity/trivy) — with **no local Trivy installation required**.

Supports both **Docker** and **Podman** as the container runtime.

## What it does

`Docker Image Scan` automates the full container security workflow:

1. **Detect** the available container runtime (Docker or Podman)
2. **Build** the image from a Dockerfile
3. **Scan** for vulnerabilities using Trivy (pulled and run as a container)
4. **Report** findings with severity breakdown, CVE details, and fix recommendations
5. **Clean up** both the scanned image and the Trivy container image

Claude analyzes the raw scan results and provides actionable recommendations — base image upgrades, package fixes, and Dockerfile improvements.

## Prerequisites

One of the following container runtimes:

- [Docker](https://docs.docker.com/get-docker/)
- [Podman](https://podman.io/docs/installation)

That's it. Trivy is automatically pulled and run as a container — no separate installation needed.

## Installation

Add the marketplace and install:

```
/plugin marketplace add igor-koricanac/claude-image-scan
/plugin install image-scan@image-scan-marketplace
```

## Usage

### Scan a Dockerfile

```
/image-scan:image-scan ./Dockerfile
```

### Scan a Dockerfile in a subdirectory

```
/image-scan:image-scan ./docker/Dockerfile
```

## Example output

```
Container runtime detected: docker

Building image from ./Dockerfile...
Successfully built claude-scan-1710000000

Vulnerability Report for claude-scan-1710000000

| Severity | Count |
|----------|-------|
| CRITICAL | 2     |
| HIGH     | 5     |
| MEDIUM   | 12    |
| LOW      | 31    |

Critical Findings:
  CVE-2023-44487  curl 7.88.1  → Fix: upgrade to 8.4.0
  CVE-2023-38545  libssl3 3.0  → Fix: upgrade to 3.0.12

Recommendations:
  - Upgrade base image to nginx:1.27-alpine
  - Update curl and openssl packages

Cleanup complete: removed scanned image and Trivy image.
```

## How it works

The plugin is a Claude Code skill defined in `skills/image-scan/SKILL.md`. When invoked:

1. Detects whether Docker or Podman is installed (prefers Docker if both are available)
2. Builds your container image from the provided Dockerfile
3. Pulls `aquasec/trivy:latest` and runs it as a container to scan the built image
4. Analyzes the JSON scan results and presents a structured report
5. Removes both the built image and the `aquasec/trivy:latest` image

No data leaves your machine — everything runs locally via your container runtime.

## Privacy

This plugin runs entirely on your local machine. It does not collect, transmit, store, or share any user data.

- No telemetry
- No analytics
- No external API calls (other than your container runtime pulling the `aquasec/trivy:latest` image from its configured registry, which the user controls)
- No network traffic initiated by the plugin itself
- Scan results are written to a local temporary directory that is deleted after the scan completes
- Built images and the Trivy container image are removed from your local machine after the scan

Because everything is local, there is no data collection to disclose.

## License

MIT
