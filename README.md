# dstack-vllm

Custom vLLM image with dstack runner dependencies for faster cold starts.

Based on `vllm/vllm-openai:latest` with pre-installed dstack runner prerequisites that eliminate the 2-4 minute `apt-get install openssh-server` delay at pod start.

## What's included

- **openssh-server** — dstack's shim checks `exists sshd` and skips `apt-get install` if found
- **sudo** — required by dstack's user setup
- **dstack user** (uid/gid 1000) — pre-created with passwordless sudo
- **dstack directories** — `/dstack/run` and `/run/sshd`

## Usage

In a dstack service config:

```yaml
type: service
name: my-llm
image: ghcr.io/plumelo/dstack-vllm:latest
commands:
  - |
    vllm serve $MODEL_ID --host 0.0.0.0 --port 8000 ...
```

## Cold start improvement

| Phase | What | Time saved |
|-------|------|-----------|
| This image | Pre-installed sshd/sudo | ~2-4 min |
| Phase 2 (planned) | B2 compilation cache | ~2-3 min more |

## CI/CD

- **Push to main**: builds and pushes `ghcr.io/plumelo/dstack-vllm:latest` + SHA tag
- **PR**: builds and runs verification checks (sshd, sudo, dstack user)