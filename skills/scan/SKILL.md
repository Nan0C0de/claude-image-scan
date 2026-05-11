---
name: scan
description: Build a container image from a Dockerfile, scan it for security vulnerabilities using Trivy (via Docker or Podman), and clean up all images afterward.
---

# Container Image Security Scanner

Scan container images for vulnerabilities using Trivy. This skill handles the full lifecycle: build, scan, report, and clean up. Trivy runs as a container — no local Trivy installation required. Supports both Docker and Podman as the container runtime.

## Input

The user provides a **path to a Dockerfile**. The input is passed via `$ARGUMENTS`.

Examples:
- `/docker-image-scan:scan ./Dockerfile`
- `/docker-image-scan:scan ./docker/Dockerfile`
- `/docker-image-scan:scan /absolute/path/to/Dockerfile`

## Workflow

Follow these steps precisely:

### Step 1 — Validate input

- The argument must be a path to a Dockerfile (contains `/` or ends in `Dockerfile` or the file exists).
- Verify the Dockerfile exists at the given path. If it does not, inform the user and stop.
- If the argument is unclear or missing, ask the user to provide a path to a Dockerfile.

### Step 2 — Detect container runtime

Check for Docker or Podman. Use whichever is available. If both are available, prefer Docker.

```bash
if command -v docker >/dev/null 2>&1; then
  RUNTIME="docker"
  echo "runtime: docker"
elif command -v podman >/dev/null 2>&1; then
  RUNTIME="podman"
  echo "runtime: podman"
else
  echo "runtime: MISSING"
fi
```

If neither is found, inform the user and stop. Provide installation links for both options:
- Docker: https://docs.docker.com/get-docker/
- Podman: https://podman.io/docs/installation

Do not proceed until at least one runtime is available.

Store the detected runtime in the `RUNTIME` variable and use `$RUNTIME` in place of `docker` for **all** subsequent commands.

### Step 3 — Build the container image

Generate a unique tag to avoid collisions:

```bash
SCAN_TAG="claude-scan-$(date +%s)"
DOCKERFILE_PATH="<user-provided-path>"
BUILD_CONTEXT="$(dirname "$DOCKERFILE_PATH")"

$RUNTIME build -t "$SCAN_TAG" -f "$DOCKERFILE_PATH" "$BUILD_CONTEXT"
```

If the build fails, show the error output to the user and stop. Do not proceed to scanning.

### Step 4 — Scan with Trivy via container

Create a temporary directory for scan results and the exported image:

```bash
SCAN_DIR=$(mktemp -d)
RESULTS_FILE="$SCAN_DIR/trivy-results.json"
```

The approach differs by runtime:

**Docker:**

Docker allows Trivy to access local images via the Docker socket. Write results to a mounted file:

```bash
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v "$SCAN_DIR:/output" \
  aquasec/trivy:latest image \
  --format json \
  --output /output/trivy-results.json \
  --severity CRITICAL,HIGH,MEDIUM,LOW \
  "$SCAN_TAG"
```

**Podman:**

Podman does not use a socket. Export the built image as a tar archive and pass it to Trivy with `--input`:

```bash
podman save -o "$SCAN_DIR/image.tar" "$SCAN_TAG"

podman run --rm \
  -v "$SCAN_DIR:/scan" \
  aquasec/trivy:latest image \
  --input /scan/image.tar \
  --format json \
  --output /scan/trivy-results.json \
  --severity CRITICAL,HIGH,MEDIUM,LOW
```

After the scan, the results are available at `$RESULTS_FILE`. If the scan fails, still proceed to cleanup (Step 6) before reporting the error.

### Step 5 — Read and analyze results

Read the full contents of `$RESULTS_FILE` **before any cleanup happens**. Parse the JSON and extract all vulnerability data you need for the report. You must have all the data in memory before proceeding to the next step.

### Step 6 — Clean up

Remove the scanned image, the Trivy container image, and the temporary directory:

```bash
$RUNTIME rmi "$SCAN_TAG" 2>/dev/null
$RUNTIME rmi aquasec/trivy:latest 2>/dev/null
rm -rf "$SCAN_DIR"
```

This step **must always run**, even if the scan fails. Wrap the scan in a way that cleanup runs regardless. Only run this step **after** you have fully read the results file in Step 5.

### Step 7 — Present the report

Now that cleanup is done, present the scan findings to the user:

**Summary table:**
| Severity | Count |
|----------|-------|
| CRITICAL | N     |
| HIGH     | N     |
| MEDIUM   | N     |
| LOW      | N     |

**Critical and High findings** — list each with:
- Package name and installed version
- Vulnerability ID (CVE)
- Fixed version (if available)
- Brief description

**Recommendations:**
- Suggest base image upgrades if the base image has known vulnerabilities
- Suggest package updates for fixable vulnerabilities
- Highlight any vulnerabilities with known exploits
- Suggest specific Dockerfile changes based on the provided Dockerfile

## Important rules

- Always detect the container runtime first and use it consistently for all commands.
- Always clean up both images (the built image and `aquasec/trivy:latest`) and the temporary directory, even if the scan fails.
- Only Docker or Podman is required — Trivy is pulled and run as a container automatically.
- If Trivy finds zero vulnerabilities, congratulate the user and confirm the image is clean.
- Keep the report concise — group LOW severity findings into a count rather than listing each one individually.
- If the scan output is extremely large, focus on CRITICAL and HIGH severity items and summarize the rest.
