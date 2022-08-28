# syntax = docker/dockerfile:experimental

ARG BASE_IMAGE=nvidia/cuda:11.5.2-cudnn8-devel-ubuntu20.04
ARG PYTHON_VERSION=3.8

FROM ${BASE_IMAGE} AS conda

ENV DEBIAN_FRONTEND noninteractive
RUN --mount=type=cache,id=apt-dev,target=/var/cache/apt \
    apt-get update && \
    apt-get install -y --no-install-recommends \
        sudo \
        ca-certificates \
        curl && \
    rm -rf /var/lib/apt/lists/*

# create a non-root user
ARG USER_ID=1000
RUN useradd -m --no-log-init --system  --uid ${USER_ID} user -g sudo
RUN echo '%sudo ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers
USER user
WORKDIR /home/user

ARG PYTHON_VERSION=3.8
RUN curl -fsSL -v -o ~/miniconda.sh -O https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh  && \
    chmod +x ~/miniconda.sh && \
    ~/miniconda.sh -b -p /home/user/miniconda && \
    rm ~/miniconda.sh && \
    /home/user/miniconda/bin/conda install -y python=${PYTHON_VERSION} && \
    /home/user/miniconda/bin/conda clean -ya
ENV PATH /home/user/miniconda/bin:$PATH
RUN echo '. /home/user/miniconda/etc/profile.d/conda.sh && conda activate base' >> /home/user/.bashrc


FROM conda AS builder

RUN conda install astunparse numpy ninja pyyaml setuptools cmake cffi typing_extensions \
                  future six requests dataclasses mkl mkl-include && \
    conda install -c pytorch magma-cuda115 && \
    conda clean -ya

ENV DEBIAN_FRONTEND noninteractive
RUN --mount=type=cache,id=apt-dev,target=/var/cache/apt \
    sudo apt-get update && \
    sudo apt-get install -y --no-install-recommends \
        ccache \
        wget \
        git \
        libnuma-dev \
        libjpeg-dev \
        libpng-dev && \
    sudo rm -rf /var/lib/apt/lists/*
RUN /usr/sbin/update-ccache-symlinks
RUN sudo mkdir /opt/ccache && ccache --set-config=cache_dir=/opt/ccache

ARG PYTORCH_VERSION=1.11.0

RUN git clone https://github.com/pytorch/pytorch && \
    cd pytorch && \
    git checkout v${PYTORCH_VERSION} && \
    git submodule sync && \
    git submodule update --init --recursive --jobs 0

WORKDIR /home/user/pytorch

ARG PYTORCH_BUILD_VERSION=${PYTORCH_VERSION}
ARG PYTORCH_BUILD_NUMBER=1
ARG TORCH_CUDA_ARCH_LIST="6.0 6.1 7.0+PTX 8.0"
ARG TORCH_NVCC_FLAGS="-Xfatbin -compress-all"

RUN --mount=type=cache,target=/opt/ccache \
    sudo chown user /opt/ccache & \
    TORCH_CUDA_ARCH_LIST=${TORCH_CUDA_ARCH_LIST} \
    TORCH_NVCC_FLAGS=${TORCH_NVCC_FLAGS} \
    CMAKE_PREFIX_PATH="$(dirname $(which conda))/../" \
    PYTORCH_BUILD_VERSION=${PYTORCH_BUILD_VERSION} \
    PYTORCH_BUILD_NUMBER=${PYTORCH_BUILD_NUMBER} \
    python setup.py bdist_wheel && \
    rm -rf build

RUN pip install /home/user/pytorch/dist/*.whl

ARG TORCHVISION_VERSION=0.12.0

RUN git clone https://github.com/pytorch/vision.git /home/user/vision && \
    cd /home/user/vision && \
    git checkout v${TORCHVISION_VERSION}

WORKDIR /home/user/vision

RUN python setup.py bdist_wheel && \
    rm -rf build

RUN pip install /home/user/vision/dist/*.whl


FROM conda AS runtime

ENV DEBIAN_FRONTEND noninteractive
RUN --mount=type=cache,id=apt-dev,target=/var/cache/apt \
    sudo apt-get update && \
    sudo apt-get install -y --no-install-recommends \
        libnuma-dev \
        wget \
        git \
        vim \
        libjpeg-dev \
        libpng-dev && \
    sudo rm -rf /var/lib/apt/lists/*

RUN conda install numpy typing_extensions mkl mkl-include cmake && \
    conda install -c pytorch magma-cuda115 && \
    conda clean -ya

RUN --mount=from=builder,src=/home/user/pytorch,dst=/pytorch \
    pip install --no-cache-dir /pytorch/dist/*.whl

RUN --mount=from=builder,src=/home/user/vision,dst=/vision \
    pip install --no-cache-dir /vision/dist/*.whl

ARG TORCH_DATA_VERSION=0.3.0

RUN pip install --no-cache-dir torchdata==${TORCH_DATA_VERSION}
