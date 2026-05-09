FROM nvidia/cuda:13.2.1-devel-ubuntu24.04 AS build

RUN apt-get update && DEBIAN_FRONTEND=noninteractive apt-get install -y \
      gcc-14 g++-14 build-essential cmake git python3 libssl-dev && \
    rm -rf /var/lib/apt/lists/*

ENV CC=gcc-14 CXX=g++-14 CUDAHOSTCXX=g++-14

WORKDIR /app
RUN git clone https://github.com/ggml-org/llama.cpp.git . && \
    git fetch origin pull/22673/head:mtp-pr && \
    git checkout mtp-pr

WORKDIR /app
RUN git clone https://github.com/ggml-org/llama.cpp.git . && \
    git fetch origin pull/22673/head:mtp-pr && \
    git checkout mtp-pr

RUN ln -s /usr/local/cuda/lib64/stubs/libcuda.so /usr/local/lib/libcuda.so.1 && \
    ln -s /usr/local/cuda/lib64/stubs/libcuda.so /usr/local/lib/libcuda.so && \
    cmake -B build \
      -DGGML_CUDA=ON \
      -DGGML_NATIVE=OFF \
      -DCMAKE_CUDA_ARCHITECTURES="86;89;90;120" \
      -DCMAKE_BUILD_TYPE=Release \
      -DGGML_CUDA_GRAPHS=ON \
      -DLLAMA_BUILD_TESTS=OFF && \
    cmake --build build --config Release --target llama-server -j$(nproc)

FROM nvidia/cuda:13.2.1-runtime-ubuntu24.04

RUN apt-get update && DEBIAN_FRONTEND=noninteractive apt-get install -y \
      openssh-server sudo curl unzip libgomp1 && \
    sed -i "s/.*PasswordAuthentication.*/PasswordAuthentication no/g" /etc/ssh/sshd_config && \
    mkdir /run/sshd && \
    rm -f /etc/ssh/ssh_host_* && \
    if id 1000 >/dev/null 2>&1; then \
      EXISTING_USER=$(id -nu 1000); \
      EXISTING_GROUP=$(getent group 1000 | cut -d: -f1); \
      usermod -l dstack -d /home/dstack -m "$EXISTING_USER"; \
      groupmod -n dstack "$EXISTING_GROUP"; \
      usermod -aG sudo dstack; \
    else \
      groupadd -g 1000 dstack && \
      useradd -u 1000 -g 1000 -G sudo -s /bin/bash -m dstack; \
    fi && \
    echo 'dstack ALL=(ALL) NOPASSWD: ALL' > /etc/sudoers.d/dstack && \
    mkdir -p /dstack/run && chmod a+rwx /dstack/run && \
    curl -fsSL https://rclone.org/install.sh | bash && \
    rm -rf /var/lib/apt/lists/*

COPY --from=build /app/build/bin/llama-server /usr/local/bin/llama-server
COPY --from=build /app/build/lib /usr/local/lib

ENV LD_LIBRARY_PATH=/usr/local/lib:$LD_LIBRARY_PATH

EXPOSE 8000