# B2 CUDA Graph Cache — Implementation Plan

## Problem

When vLLM starts, it compiles CUDA graphs specific to the GPU architecture. This takes 2–5 minutes and runs on every cold start or re-provision. The compiled graphs are stored in `VLLM_CACHE_ROOT` (defaults to `/root/.cache/vllm` or `/workspace/vllm_cache` depending on the image).

## Goal

Cache compiled CUDA graphs in Backblaze B2 so that subsequent starts on the same GPU architecture + vLLM version pull a pre-compiled cache instead of recompiling from scratch. Target: save ~2–3 minutes per cold start.

## Architecture

A single shared `entrypoint.sh` script wraps the vLLM command:

```
entrypoint.sh
  ├── If B2_BUCKET + B2_ACCOUNT_ID + B2_APPLICATION_KEY are set:
  │   ├── Detect CUDA arch via nvidia-smi → sm_86, sm_89, etc.
  │   ├── Detect vLLM version via python
  │   ├── rclone copy b2:<bucket>/vllm_cache/<arch>/<version>/ → $VLLM_CACHE_ROOT
  │   └── On failure: warn and proceed (don't block startup)
  ├── exec "$@" as child process (the vLLM command)
  │   └── Forward SIGTERM/SIGINT to child
  ├── Wait for vLLM to exit
  └── If B2 is configured: rclone copy $VLLM_CACHE_ROOT → b2:<bucket>/vllm_cache/<arch>/<version>/
      └── Timeout: B2_UPLOAD_TIMEOUT seconds (default 120)
```

## File Structure

```
images/
  entrypoint.sh              ← single shared script
  vllm/
    Dockerfile               ← COPY ../entrypoint.sh, ENTRYPOINT
  vllm-nightly/
    Dockerfile               ← same pattern
```

Docker build context must be `./images` (not `./images/vllm`) so that `COPY ../entrypoint.sh` resolves. Symlinks in `images/vllm/entrypoint.sh → ../entrypoint.sh` are for local editing convenience only; Docker COPY resolves real paths from the build context.

## B2 Cache Key Structure

```
<bucket>/vllm_cache/sm_86/0.6.3/
<bucket>/vllm_cache/sm_89/0.6.3/
<bucket>/vllm_cache/sm_90/0.7.0/
```

Keyed by **both** CUDA arch and vLLM version to prevent stale cache issues across GPU generations or vLLM upgrades.

## Environment Variables

| Variable | Required | Default | Description |
|---|---|---|---|
| `B2_BUCKET` | Yes | — | B2 bucket name |
| `B2_ACCOUNT_ID` | Yes | — | B2 account ID (keyID) |
| `B2_APPLICATION_KEY` | Yes | — | B2 application key |
| `B2_UPLOAD_TIMEOUT` | No | `120` | Max seconds for upload on shutdown |

All three B2 credentials must be set to enable caching. If any are missing, the entrypoint degrades to a plain `exec "$@"`.

## Entrypoint Script (`images/entrypoint.sh`)

```bash
#!/usr/bin/env bash
set -euo pipefail

VLLM_CACHE_ROOT="${VLLM_CACHE_ROOT:-/root/.cache/vllm}"
B2_UPLOAD_TIMEOUT="${B2_UPLOAD_TIMEOUT:-120}"

b2_enabled() {
  [[ -n "${B2_BUCKET:-}" && -n "${B2_ACCOUNT_ID:-}" && -n "${B2_APPLICATION_KEY:-}" ]]
}

detect_cuda_arch() {
  if command -v nvidia-smi &>/dev/null; then
    local cap
    cap=$(nvidia-smi --query-gpu=compute_cap --format=csv,noheader,nounits 2>/dev/null | head -1 | tr -d ' .')
    if [[ -n "$cap" ]]; then
      echo "sm_${cap}"
      return
    fi
  fi
  echo "unknown"
}

detect_vllm_version() {
  python3 -c "import vllm; print(vllm.__version__)" 2>/dev/null || echo "unknown"
}

setup_rclone() {
  export RCLONE_CONFIG_B2_TYPE="s3"
  export RCLONE_CONFIG_B2_PROVIDER="Other"
  export RCLONE_CONFIG_B2_ENV_AUTH="false"
  export RCLONE_CONFIG_B2_ENDPOINT="s3.us-east-005.backblazeb2.com"
  export RCLONE_CONFIG_B2_ACCESS_KEY_ID="${B2_ACCOUNT_ID}"
  export RCLONE_CONFIG_B2_SECRET_ACCESS_KEY="${B2_APPLICATION_KEY}"
  export RCLONE_CONFIG_B2_FORCE_PATH_STYLE="true"
}

pull_cache() {
  local arch version remote_path
  arch=$(detect_cuda_arch)
  version=$(detect_vllm_version)
  remote_path="b2:${B2_BUCKET}/vllm_cache/${arch}/${version}"

  echo "[b2-cache] Pulling cache from ${remote_path}"

  if rclone copy "${remote_path}/" "${VLLM_CACHE_ROOT}/" \
    --progress \
    --transfers=4 \
    --checkers=8 \
    --contimeout=30s \
    --timeout=60s \
    --retries=2 \
    --low-level-retries=1 2>&1; then
    echo "[b2-cache] Cache restored successfully"
  else
    echo "[b2-cache] WARN: Failed to pull cache, vLLM will compile fresh" >&2
  fi
}

push_cache() {
  local arch version remote_path
  arch=$(detect_cuda_arch)
  version=$(detect_vllm_version)
  remote_path="b2:${B2_BUCKET}/vllm_cache/${arch}/${version}"

  if [[ ! -d "${VLLM_CACHE_ROOT}" ]] || [[ -z "$(ls -A "${VLLM_CACHE_ROOT}" 2>/dev/null)" ]]; then
    echo "[b2-cache] No cache to upload, skipping"
    return
  fi

  echo "[b2-cache] Uploading cache to ${remote_path}"

  timeout "${B2_UPLOAD_TIMEOUT}" rclone copy "${VLLM_CACHE_ROOT}/" "${remote_path}/" \
    --progress \
    --transfers=4 \
    --checkers=8 \
    --contimeout=30s \
    --timeout=60s \
    --retries=1 \
    --low-level-retries=1 2>&1 || \
    echo "[b2-cache] WARN: Upload failed or timed out (best-effort)" >&2
}

child_pid=""

forward_signal() {
  if [[ -n "${child_pid}" ]]; then
    kill -"$1" "${child_pid}" 2>/dev/null || true
  fi
}

trap 'forward_signal TERM' TERM
trap 'forward_signal INT' INT

# --- Main ---

if b2_enabled; then
  setup_rclone
  pull_cache
else
  echo "[b2-cache] B2 not configured, skipping cache pull"
fi

echo "[b2-cache] Starting: $*"
"$@" &
child_pid=$!
wait "${child_pid}"
exit_code=$?

if b2_enabled; then
  push_cache
fi

exit "${exit_code}"
```

## Dockerfile Changes

Both `images/vllm/Dockerfile` and `images/vllm-nightly/Dockerfile` need:

```dockerfile
RUN curl -fsSL https://downloads.rclone.org/rclone-current-linux-amd64.deb -o /tmp/rclone.deb \
    && dpkg -i /tmp/rclone.deb \
    && rm /tmp/rclone.deb

COPY ../entrypoint.sh /usr/local/bin/entrypoint.sh
RUN chmod +x /usr/local/bin/entrypoint.sh

ENTRYPOINT ["entrypoint.sh"]
```

This requires changing the Docker build context from `./images/vllm` to `./images` in CI/CD workflows so that `COPY ../entrypoint.sh` resolves correctly.

## Workflow Changes

### `.github/workflows/build.yml`

For both `build-vllm` and `build-vllm-nightly` jobs, change the `docker/build-push-action` context:

```yaml
- name: Build and push Docker image
  uses: docker/build-push-action@v7
  with:
    context: ./images
    build-contexts: 
      vllm=./images/vllm
    file: ./images/vllm/Dockerfile
    push: true
    tags: ${{ steps.meta.outputs.tags }}
    labels: ${{ steps.meta.outputs.labels }}
```

Similarly for vllm-nightly, point to `file: ./images/vllm-nightly/Dockerfile`.

### `.github/workflows/ci.yml`

Update build commands and add verification steps:

```yaml
- name: Build vllm image
  run: docker build -t test-vllm -f ./images/vllm/Dockerfile ./images

- name: Verify openssh-server installed
  run: docker run --rm test-vllm which sshd

- name: Verify sudo installed
  run: docker run --rm test-vllm which sudo

- name: Verify rclone installed
  run: docker run --rm test-vllm which rclone

- name: Verify entrypoint exists
  run: docker run --rm test-vllm test -x /usr/local/bin/entrypoint.sh
```

Same pattern for vllm-nightly.

## Graceful Degradation

| Condition | Behavior |
|---|---|
| B2 credentials not set | Skip entirely, `exec "$@"` directly |
| nvidia-smi unavailable | CUDA arch = `unknown`, pull still attempted (may hit nothing) |
| B2 unreachable on pull | Warn and proceed, vLLM compiles fresh |
| B2 unreachable on push | Warn and exit, no delay |
| SIGTERM during vLLM run | Forwarded to vLLM, upload runs after exit |
| SIGKILL | No cleanup (expected — OS-level kill) |
| Upload exceeds B2_UPLOAD_TIMEOUT | Terminated, best-effort |

## Usage Examples

### With B2 caching enabled

```yaml
type: service
name: my-vllm
image: ghcr.io/plumelo/dstack-images:vllm-latest
env:
  - B2_BUCKET=my-vllm-cache
  - B2_ACCOUNT_ID=005xxxxx00000000000000002
  - B2_APPLICATION_KEY=K005xxxxxxxxxxxxxxxxxxxxxxxxxxxxx
  - B2_UPLOAD_TIMEOUT=180
commands:
  - vllm serve $MODEL_ID --host 0.0.0.0 --port 8000
```

### Without B2 caching (default behavior)

```yaml
type: service
name: my-vllm
image: ghcr.io/plumelo/dstack-images:vllm-latest
commands:
  - vllm serve $MODEL_ID --host 0.0.0.0 --port 8000
```

No B2 env vars = no caching, vLLM compiles on every start as it does today.

## Rclone B2 Endpoint Notes

The S3-compatible endpoint `s3.us-east-005.backblazeb2.com` is used because rclone's native B2 backend doesn't support directory-style operations as cleanly. If your B2 bucket is in a different region, adjust `RCLONE_CONFIG_B2_ENDPOINT` accordingly.

Alternative: Use rclone's native B2 backend by setting `RCLONE_CONFIG_B2_TYPE=b2` instead of `s3`. This requires the `RCLONE_CONFIG_B2_ACCOUNT` and `RCLONE_CONFIG_B2_KEY` fields instead of S3-style auth. The S3-compatible approach was chosen for simplicity since it uses the same credential pair (`B2_ACCOUNT_ID` + `B2_APPLICATION_KEY`) that users already have from the B2 console.

## Implementation Checklist

- [ ] Create `images/entrypoint.sh`
- [ ] Update `images/vllm/Dockerfile` — add rclone + entrypoint
- [ ] Update `images/vllm-nightly/Dockerfile` — add rclone + entrypoint
- [ ] Update `.github/workflows/build.yml` — change build context
- [ ] Update `.github/workflows/ci.yml` — change build context + add verification steps
- [ ] Update `README.md` — document B2 env vars, usage, mark Phase 2 as done
- [ ] Test with a real B2 bucket