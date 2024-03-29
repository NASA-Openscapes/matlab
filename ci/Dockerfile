# Copyright 2023 The MathWorks, Inc.
# Builds Docker image with 
# 1. MATLAB - Using MPM
# 2. MATLAB Integration for Jupyter
# on a base image of jupyter/base-notebook:ubuntu-22.04

## Sample Build Command:
# docker build --build-arg MATLAB_RELEASE=r2023a \
#              --build-arg MATLAB_PRODUCT_LIST="MATLAB Deep_Learning_Toolbox Symbolic_Math_Toolbox"\
#              --build-arg LICENSE_SERVER=12345@hostname.com \
#              -t my_matlab_image_name .

# Specify release of MATLAB to build. (use lowercase, default is r2023a)
ARG MATLAB_RELEASE=r2023a

# Specify the list of products to install into MATLAB, 
ARG MATLAB_PRODUCT_LIST="MATLAB Deep_Learning_Toolbox Image_Processing_Toolbox Statistics_and_Machine_Learning_Toolbox Simulink"
# Optional Network License Server information
ARG LICENSE_SERVER

# If LICENSE_SERVER is provided then SHOULD_USE_LICENSE_SERVER will be set to "_use_lm"
ARG SHOULD_USE_LICENSE_SERVER=${LICENSE_SERVER:+"_with_lm"}

# Default DDUX information
ARG MW_CONTEXT_TAGS=MATLAB_PROXY:JUPYTER:MPM:V1


## jupyter/base-notebook:python-3.10 is based on Ubuntu 22.04 and is being used as,
## the matlabengineforpython for R2023a only supports upto python v3.10
# Base Jupyter image without LICENSE_SERVER
FROM jupyter/base-notebook:python-3.10 AS base_jupyter_image

# Base Jupyter image with LICENSE_SERVER
FROM jupyter/base-notebook:python-3.10 AS base_jupyter_image_with_lm
ARG LICENSE_SERVER
# If license server information is available, then use it to set environment variable
ENV MLM_LICENSE_FILE=${LICENSE_SERVER}

# Select base Jupyter image based on whether LICENSE_SERVER is provided
FROM base_jupyter_image${SHOULD_USE_LICENSE_SERVER}
ARG MW_CONTEXT_TAGS
ARG MATLAB_RELEASE
ARG MATLAB_PRODUCT_LIST

# Switch to root user
USER root
ENV DEBIAN_FRONTEND="noninteractive" TZ="Etc/UTC"

# List of MATLAB Dependencies for Ubuntu 22.04 and MATLAB R2023a
ARG MATLAB_DEPS_REQUIREMENTS="ca-certificates libasound2 libc6 libcairo-gobject2 libcairo2 libcap2 libcrypt1 libcups2 libdrm2 libgbm1 libgdk-pixbuf-2.0-0 libgl1 libglib2.0-0 libgstreamer-plugins-base1.0-0 libgstreamer1.0-0 libgtk-3-0 libice6 libnspr4 libnss3 libodbc2 libodbcinst2 libpam0g libpango-1.0-0 libpangocairo-1.0-0 libpangoft2-1.0-0 libsndfile1 libuuid1 libwayland-client0 libxcomposite1 libxcursor1 libxdamage1 libxfixes3 libxft2 libxinerama1 libxrandr2 libxt6 libxtst6 libxxf86vm1 locales locales-all make net-tools procps sudo unzip zlib1g"
ARG MATLAB_DEPS_REQUIREMENTS_FILE_NAME="matlab-deps-${MATLAB_RELEASE}-base-dependencies.txt"

## Install dependencies
## MATLAB versions older than 22b need libpython3.9 which is only present in the deadsnakes PPA on ubuntu:22.04
RUN echo ${MATLAB_DEPS_REQUIREMENTS} > ${MATLAB_DEPS_REQUIREMENTS_FILE_NAME} \
    && apt-get update \
    && export isJammy=`cat /etc/lsb-release | grep DISTRIB_RELEASE=22.04 | wc -l` \
    && export needsPy39=`cat ${MATLAB_DEPS_REQUIREMENTS_FILE_NAME} | grep libpython3.9 | wc -l` \
    && if [[ isJammy -eq 1 && needsPy39 -eq 1 ]] ; then apt-get install -y software-properties-common && add-apt-repository ppa:deadsnakes/ppa ; fi \
    && xargs -a ${MATLAB_DEPS_REQUIREMENTS_FILE_NAME} -r apt-get install --no-install-recommends -y \
    unzip \
    ca-certificates \
    xvfb \
    git \
    && apt-get clean \
    && apt-get -y autoremove \
    && rm -rf /var/lib/apt/lists/* ${MATLAB_DEPS_REQUIREMENTS_FILE_NAME}

# Note on glibc patch - See https://github.com/mathworks/build-glibc-bz-19329-patch
# Installation is skipped on Ubuntu 22.04 (Jammy) as it already contains the glibc fix

# Run mpm to install MATLAB in the target location and delete the mpm installation afterwards
RUN wget -q https://www.mathworks.com/mpm/glnxa64/mpm && \ 
    chmod +x mpm && \
    ./mpm install \
    --release=${MATLAB_RELEASE} \
    --destination=/opt/matlab \
    --products ${MATLAB_PRODUCT_LIST} && \
    rm -f mpm /tmp/mathworks_root.log && \
    ln -s /opt/matlab/bin/matlab /usr/local/bin/matlab

# Optional: Install MATLAB Engine for Python, if possible. 
# Note: Failure to install does not stop the build.
RUN apt-get update \
    && apt-get install --no-install-recommends -y  python3-distutils \
    && apt-get clean \
    && apt-get -y autoremove \
    && rm -rf /var/lib/apt/lists/* \
    && cd /opt/matlab/extern/engines/python \
    && python setup.py install 

# Switch back to notebook user
USER $NB_USER

# Install integration
RUN python -m pip install jupyter-matlab-proxy

# Make JupyterLab the default environment
ENV JUPYTER_ENABLE_LAB="yes"

ENV MW_CONTEXT_TAGS=${MW_CONTEXT_TAGS}

## Conda environment for Openscapes

ENV CONDA_ENV="openscapes"
ENV PYTHON_PREFIX=/opt/conda

COPY . /tmp/build
WORKDIR /tmp/build

RUN echo "Checking for 'conda-linux-64.lock' or 'environment.yml'..." \
        ; if test -f "conda-linux-64.lock" ; then \
        conda create --name ${CONDA_ENV} --file conda-linux-64.lock \
        ; elif test -f "environment.yml" ; then \
        conda env create --name ${CONDA_ENV} -f environment.yml  \
        ; else echo "No conda-linux-64.lock or environment.yml! *creating default env*" ; \
        conda create --name ${CONDA_ENV} pangeo-notebook \
        ; fi \
        && conda clean -yaf \
        && find ${CONDA_DIR} -follow -type f -name '*.a' -delete \
        && find ${CONDA_DIR} -follow -type f -name '*.pyc' -delete \
        && find ${CONDA_DIR} -follow -type f -name '*.js.map' -delete \
        ; if [ -d ${NB_PYTHON_PREFIX}/lib/python*/site-packages/bokeh/server/static ]; then \
        find ${NB_PYTHON_PREFIX}/lib/python*/site-packages/bokeh/server/static -follow -type f -name '*.js' ! -name '*.min.js' -delete \
        ; fi

WORKDIR /home/${NB_USER}
# Install integration
EXPOSE 8888
