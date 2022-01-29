FROM public.ecr.aws/j1r0q0g6/notebooks/notebook-servers/codeserver:v1.3.1-rc.0
USER root
# args - software versions
ARG CODESERVER_PYTHON_VERSION=2021.5.842923320
ARG MINIFORGE_ARCH="x86_64"
# renovate: datasource=github-tags depName=conda-forge/miniforge versioning=loose
ARG MINIFORGE_VERSION=4.10.1-4
ARG PIP_VERSION=21.1.2
ARG PYTHON_VERSION=3.6.9

# setup environment for conda
ENV CONDA_DIR /opt/conda
ENV PATH "${CONDA_DIR}/bin:${PATH}"
RUN mkdir -p ${CONDA_DIR} \
    && chown -R ${NB_USER}:users ${CONDA_DIR}

# install - conda, pip, python
RUN curl -sL "https://github.com/conda-forge/miniforge/releases/download/${MINIFORGE_VERSION}/Miniforge3-${MINIFORGE_VERSION}-Linux-${MINIFORGE_ARCH}.sh" -o /tmp/Miniforge3.sh \
    && curl -sL "https://github.com/conda-forge/miniforge/releases/download/${MINIFORGE_VERSION}/Miniforge3-${MINIFORGE_VERSION}-Linux-${MINIFORGE_ARCH}.sh.sha256" -o /tmp/Miniforge3.sh.sha256 \
    && echo "$(cat /tmp/Miniforge3.sh.sha256 | awk '{ print $1; }') /tmp/Miniforge3.sh" | sha256sum --check \
    && rm /tmp/Miniforge3.sh.sha256 \
    && /bin/bash /tmp/Miniforge3.sh -b -f -p ${CONDA_DIR} \
    && rm /tmp/Miniforge3.sh \
    && conda config --system --set auto_update_conda false \
    && conda config --system --set show_channel_urls true \
    && echo "conda ${MINIFORGE_VERSION:0:-2}" >> ${CONDA_DIR}/conda-meta/pinned \
    && echo "python ${PYTHON_VERSION}" >> ${CONDA_DIR}/conda-meta/pinned \
    && conda install -y -q \
    python=${PYTHON_VERSION} \
    conda=${MINIFORGE_VERSION:0:-2} \
    pip=${PIP_VERSION} \
    && conda update -y -q --all \
    && conda clean -a -f -y \
    && chown -R ${NB_USER}:users ${CONDA_DIR} \
    && chown -R ${NB_USER}:users ${HOME}

# install - requirements.txt
COPY --chown=jovyan:users requirements.txt /tmp
RUN python3 -m pip install -r /tmp/requirements.txt --quiet --no-cache-dir \
    && rm -f /tmp/requirements.txt \
    && chown -R ${NB_USER}:users ${CONDA_DIR} \
    && chown -R ${NB_USER}:users ${HOME}

# install - codeserver extensions
RUN code-server --install-extension "ms-python.python@${CODESERVER_PYTHON_VERSION}"

# s6 - copy scripts
COPY --chown=jovyan:users s6/ /etc
# needed for LIBNVINFER
ARG OLD_CUDA_VERSION=9.0
# args - software versions
ARG CUDA_VERSION=9.0
ARG CUDA_COMPAT_VERSION=460.73.01-1
ARG CUDA_CUDART_VERSION=11.2.152-1
ARG CUDNN_VERSION=8.1.0.77-1
ARG LIBNVINFER_VERSION=7.2.3-1

# we need bash's env var character substitution
SHELL ["/bin/bash", "-c"]

# install - cuda
# for `cuda-compat-*`: https://docs.nvidia.com/cuda/eula/index.html#attachment-a
RUN curl -sL "https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2004/x86_64/7fa2af80.pub" | apt-key add - \
    && echo "deb https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2004/x86_64/ /" > /etc/apt/sources.list.d/cuda.list \
    && apt-get -yq update \
    && apt-get -yq install --no-install-recommends \
    cuda-compat-${CUDA_VERSION/./-}=${CUDA_COMPAT_VERSION} \
    cuda-cudart-${CUDA_VERSION/./-}=${CUDA_CUDART_VERSION} \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* \
    && ln -s /usr/local/cuda-${CUDA_VERSION} /usr/local/cuda

# envs - cuda
ENV PATH /usr/local/nvidia/bin:/usr/local/cuda/bin:${PATH}
ENV LD_LIBRARY_PATH /usr/local/nvidia/lib:/usr/local/nvidia/lib64
ENV NVIDIA_VISIBLE_DEVICES all
ENV NVIDIA_DRIVER_CAPABILITIES compute,utility
ENV NVIDIA_REQUIRE_CUDA "cuda>=${CUDA_VERSION}"

# install - other nvidia stuff
RUN curl -sL "https://developer.download.nvidia.com/compute/machine-learning/repos/ubuntu2004/x86_64/7fa2af80.pub" | apt-key add - \
    && echo "deb https://developer.download.nvidia.com/compute/machine-learning/repos/ubuntu2004/x86_64 /" > /etc/apt/sources.list.d/nvidia-ml.list \
    && echo "deb https://developer.download.nvidia.com/compute/machine-learning/repos/ubuntu1804/x86_64 /" > /etc/apt/sources.list.d/nvidia-ml.list \
    && apt-get -yq update \
    && apt-get -yq install --no-install-recommends \
    cm-super \
    cuda-command-line-tools-${CUDA_VERSION/./-} \
    cuda-nvrtc-${CUDA_VERSION/./-} \
    libcublas-${CUDA_VERSION/./-} \
    libcudnn8=${CUDNN_VERSION}+cuda${CUDA_VERSION} \
    libcufft-${CUDA_VERSION/./-} \
    libcurand-${CUDA_VERSION/./-} \
    libcusolver-${CUDA_VERSION/./-} \
    libcusparse-${CUDA_VERSION/./-} \
    libfreetype6-dev \
    libhdf5-serial-dev \
    libnvinfer7=${LIBNVINFER_VERSION}+cuda${OLD_CUDA_VERSION} \
    libnvinfer-plugin7=${LIBNVINFER_VERSION}+cuda${OLD_CUDA_VERSION} \
    libzmq3-dev \
    pkg-config \
    # can't be used until NVIDIA updates (requires python < 3.7)
    # python3-libnvinfer=${LIBNVINFER_VERSION}+cuda${CUDA_VERSION} \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# tensorflow fix - CUDA profiling, tensorflow requires CUPTI
ENV LD_LIBRARY_PATH /usr/local/cuda/extras/CUPTI/lib64:/usr/local/cuda/lib64:${LD_LIBRARY_PATH}

# tensorflow fix - wrong libcuda lib path (+ reconfigure dynamic linker run-time bindings)
RUN ln -s /usr/local/cuda/lib64/stubs/libcuda.so /usr/local/cuda/lib64/stubs/libcuda.so.1 \
    && echo "/usr/local/cuda/lib64/stubs" > /etc/ld.so.conf.d/z-cuda-stubs.conf \
    && ldconfig

# tensorflow fix - wrong libcusolver lib path
# https://github.com/tensorflow/tensorflow/issues/43947#issuecomment-748273679
RUN ln -s /usr/local/cuda-${CUDA_VERSION}/targets/x86_64-linux/lib/libcusolver.so.11 /usr/local/cuda-${CUDA_VERSION}/targets/x86_64-linux/lib/libcusolver.so.10

# tensorflow fix - some tensorflow tools expect a `python` binary
RUN ln -s $(which python3) /usr/local/bin/python

RUN python3 -m pip install --upgrade https://storage.googleapis.com/tensorflow/linux/gpu/tensorflow_gpu-1.12.0-cp36-cp36m-linux_x86_64.whl
COPY cuda-requirements.txt /tmp/requirements.txt
RUN python3 -m pip install -r /tmp/requirements.txt --quiet --no-cache-dir \
    && rm -f /tmp/requirements.txt
USER $NB_UID

EXPOSE 8888

ENTRYPOINT ["/init"]