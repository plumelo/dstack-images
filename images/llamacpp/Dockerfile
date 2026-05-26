FROM ghcr.io/ggml-org/llama.cpp:server-cuda

RUN apt-get update && DEBIAN_FRONTEND=noninteractive apt-get install -y \
      openssh-server sudo \
    && rm -rf /var/lib/apt/lists/*

EXPOSE 8000