# dstack-images

Custom Docker images with dstack runner dependencies for faster cold starts.

## Images

### vllm

Based on `vllm/vllm-openai:latest` with pre-installed dstack runner prerequisites that eliminate the 2-4 minute `apt-get install openssh-server` delay at pod start.

- **Tag**: `ghcr.io/plumelo/dstack-images:vllm-latest`
- **What's added**: openssh-server, sudo, dstack user (uid/gid 1000), `/dstack/run`

### vllm-nightly

Same as vllm but based on `vllm/vllm-openai:nightly` — bleeding edge vLLM builds with the latest features and fixes (may be less stable).

- **Tag**: `ghcr.io/plumelo/dstack-images:vllm-nightly-latest`
- **What's added**: openssh-server, sudo, dstack user (uid/gid 1000), `/dstack/run`

## Usage

```yaml
# vLLM (stable)
type: service
name: my-vllm
image: ghcr.io/plumelo/dstack-images:vllm-latest
commands:
  - |
    vllm serve $MODEL_ID --host 0.0.0.0 --port 8000 ...

# vLLM (nightly)
type: service
name: my-vllm
image: ghcr.io/plumelo/dstack-images:vllm-nightly-latest
commands:
  - |
    vllm serve $MODEL_ID --host 0.0.0.0 --port 8000 ...
```

## Cold start improvement

| Phase | What | Time saved |
|-------|------|-----------|
| This image | Pre-installed sshd/sudo | ~2-4 min |
| Phase 2 (planned) | B2 compilation cache (rclone) | ~2-3 min more |

## CI/CD

- **Push to main**: builds and pushes both image tags + SHA tags
- **PR**: builds and runs verification checks per image
