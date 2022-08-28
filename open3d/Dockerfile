# syntax = docker/dockerfile:experimental

ARG BASE_IMAGE=vivym/pytorch:v1.11.0-py38-cu115

FROM ${BASE_IMAGE} as builder

ENV DEBIAN_FRONTEND=noninteractive
RUN --mount=type=cache,id=apt-dev,target=/var/cache/apt \
    sudo apt-get update && \
    sudo DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
        ca-certificates \
        ccache \
        curl \
        wget \
        git \
        xorg-dev \
        libglu1-mesa-dev \
        libsdl2-dev \
        libc++-7-dev \
        libc++abi-7-dev \
        ninja-build \
        libxi-dev \
        libtbb-dev \
        libosmesa6-dev && \
    sudo rm -rf /var/lib/apt/lists/*
RUN sudo /usr/sbin/update-ccache-symlinks
RUN sudo mkdir /opt/ccache && sudo chown user /opt/ccache && ccache --set-config=cache_dir=/opt/ccache

RUN git clone https://github.com/isl-org/Open3D.git

WORKDIR /home/user/Open3D

RUN pip install --no-cache-dir yapf jupyterlab ipywidgets jupyter_packaging

RUN curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
RUN --mount=type=cache,id=apt-dev,target=/var/cache/apt \
    sudo apt-get update && \
    sudo apt-get install -y nodejs && \
    sudo rm -rf /var/lib/apt/lists/* && \
    sudo npm install -g yarn

RUN --mount=type=cache,target=/opt/ccache \
    sudo chown user /opt/ccache & \
    mkdir build && \
    cd build && \
    cmake \
        -DPython3_ROOT="$(dirname $(which conda))/../" \
        -DWITH_OPENMP=ON \
        -DWITH_SIMD=ON \
        -DBUILD_PYTORCH_OPS=ON \
        -DBUILD_CUDA_MODULE=ON \
        -DBUILD_COMMON_CUDA_ARCHS=ON \
        -DGLIBCXX_USE_CXX11_ABI=ON \
        -DBUILD_JUPYTER_EXTENSION=ON \
        .. && \
    make -j$(nproc) && \
    make pip-package && \
    cp -R lib/python_package /home/user/ && \
    cd .. && \
    rm -rf build


# FROM ${BASE_IMAGE} as runtime

# RUN  --mount=from=builder,src=/home/user/Open3D,dst=/Open3D \
#     pip install --no-cache-dir /Open3D/build/lib/python_package/pip_package/*.whl

# WORKDIR /home/user
