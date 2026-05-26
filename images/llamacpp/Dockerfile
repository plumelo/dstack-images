FROM ghcr.io/ggml-org/llama.cpp:server-cuda

RUN apt-get update && DEBIAN_FRONTEND=noninteractive apt-get install -y \
      openssh-server sudo \
    && sed -i "s/.*PasswordAuthentication.*/PasswordAuthentication no/g" /etc/ssh/sshd_config \
    && mkdir /run/sshd \
    && rm -f /etc/ssh/ssh_host_* \
    && groupadd -g 1000 dstack \
    && useradd -u 1000 -g 1000 -G sudo -s /bin/bash -m dstack \
    && echo 'dstack ALL=(ALL) NOPASSWD: ALL' > /etc/sudoers.d/dstack \
    && mkdir -p /dstack/run && chmod a+rwx /dstack/run \
    && rm -rf /var/lib/apt/lists/*

EXPOSE 8000