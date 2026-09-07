FROM ubuntu:18.04

RUN sed -i 's|http://\(archive\|security\).ubuntu.com/ubuntu/|http://old-releases.ubuntu.com/ubuntu/|g' /etc/apt/sources.list \
    && apt-get update && apt-get install -y \
    build-essential \
    libncurses-dev \
    bison \
    flex \
    libssl-dev \
    libelf-dev \
    bc \
    device-tree-compiler \
    u-boot-tools \
    lzop \
    git \
    && rm -rf /var/lib/apt/lists/*

COPY toolchains/gcc-linaro-4.9.4-2017.01-x86_64_arm-linux-gnueabihf \
     /root/tools/cross_compiler/gcc-linaro-4.9.4-2017.01-x86_64_arm-linux-gnueabihf

ENV PATH=/root/tools/cross_compiler/gcc-linaro-4.9.4-2017.01-x86_64_arm-linux-gnueabihf/bin:${PATH}

WORKDIR /workspace
