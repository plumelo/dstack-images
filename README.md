# dstack-images

Custom Docker images with dstack runner dependencies for faster cold starts.

## Images

### vllm

Based on `vllm/vllm-openai:latest` with pre-installed dstack runner prerequisites that eliminate the 2-4 minute `apt-get install openssh-server` delay at pod start.

- **Tag**: `ghcr.io/plumelo/dstack-images:vllm-latest`
- **What's added**: openssh-server, sudo, dstack user (uid/gid 1000), `/dstack/run`

### llama-mtp

Based on `nvidia/cuda:12.8.1-runtime-ubuntu24.04` with llama.cpp server built from [PR #22673](https://github.com/ggml-org/llama.cpp/pull/22673) (multi-token prediction). Includes dstack runner prerequisites and rclone (for future B2 cache).

- **Tag**: `ghcr.io/plumelo/dstack-images:llama-mtp-latest`
- **What's added**: llama-server (MTP build), openssh-server, sudo, dstack user, rclone
- **CUDA architectures**: 86, 89, 90, 120 (RTX 3090, RTX 4090, H200, RTX PRO 6000 Blackwell)

## Usage

```yaml
# vLLM
type: service
name: my-vllm
image: ghcr.io/plumelo/dstack-images:vllm-latest
commands:
  - |
    vllm serve $MODEL_ID --host 0.0.0.0 --port 8000 ...

# llama.cpp MTP
type: service
name: my-llama-mtp
image: ghcr.io/plumelo/dstack-images:llama-mtp-latest
commands:
  - |
    llama-server --hf-repo ... --hf-file ... --port 8000 --spec-type mtp --spec-draft-n-max 3 --cont-batching
```

## Cold start improvement

| Phase | What | Time saved |
|-------|------|-----------|
| vllm image | Pre-installed sshd/sudo | ~2-4 min |
| Phase 2 (planned) | B2 compilation cache (rclone) | ~2-3 min more |

## CI/CD

- **Push to main**: builds and pushes both image tags + SHA tags
- **PR**: builds and runs verification checks per image