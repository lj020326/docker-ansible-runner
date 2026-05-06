# Dockerfile for ansible-runner container
# This Dockerfile builds a custom ansible-runner container image.
# It allows specifying Ansible core and Python versions via build arguments.

# ARG instructions used in FROM must come before the FROM instruction
ARG PYTHON_VERSION="3.12"

# Stage 1: Extraction
ARG UBUNTU_VERSION=24.04
FROM ubuntu:${UBUNTU_VERSION} AS builder
RUN apt-get update && apt-get install -y unzip tar

ENV YQ_VERSION=v4.53.2
ENV YQ_BINARY=yq_linux_amd64
ARG GOVC_VERSION=v0.53.1
ARG PACKER_VERSION=1.15.3

# Copy all local files into the builder
COPY files/${YQ_BINARY}_${YQ_VERSION}.tar.gz /tmp/
COPY files/govc_${GOVC_VERSION}_linux_x86_64.tar.gz /tmp/
COPY files/packer_${PACKER_VERSION}_linux_amd64.zip /tmp/

# Extract YQ
RUN mkdir -p /out/usr/local/bin && \
    tar -xzf /tmp/${YQ_BINARY}_${YQ_VERSION}.tar.gz -C /out/usr/local/bin/ --wildcards "*${YQ_BINARY}*" && \
    mv /out/usr/local/bin/${YQ_BINARY} /out/usr/local/bin/yq

# Extract GOVC
RUN tar -xzf /tmp/govc_${GOVC_VERSION}_linux_x86_64.tar.gz -C /out/usr/local/bin/ --wildcards "*govc*"

# Extract Packer
RUN unzip /tmp/packer_${PACKER_VERSION}_linux_amd64.zip packer -d /out/usr/local/bin/

# Copy the retry script into the image
COPY retry_command.sh /out/usr/local/bin/retry_command.sh

# Make the script executable
RUN chmod +x /out/usr/local/bin/retry_command.sh

# Ensure everything is executable
RUN chmod +x /out/usr/local/bin/*

# Use a Python base image that matches the desired Python version
# We use a slim Debian-based image for smaller size.
FROM python:${PYTHON_VERSION}-slim-bullseye
ARG PYTHON_VERSION="3.12"

ARG ANSIBLE_CORE_VERSION="latest"

LABEL maintainer="Lee Johnson <ljohnson@dettonville.org>"
LABEL description="Custom Ansible Test Container for Ansible Core ${ANSIBLE_CORE_VERSION} and Python ${PYTHON_VERSION}"

ARG BUILD_DATE
ARG BUILD_ID=devel
LABEL build=$BUILD_ID

# Set environment variables
ENV container=docker
ENV DEBIAN_FRONTEND=noninteractive

# Set PIP_NO_CACHE_DIR to prevent pip from caching packages inside the image
ENV PIP_NO_CACHE_DIR=1

ENV PATH="/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin:/usr/local/ansible/bin"
ENV PATH="/root/.local/bin:${PATH}"

## ref: https://octopus.com/blog/using-ubuntu-docker-image
RUN echo 'APT::Install-Suggests "0";' >> /etc/apt/apt.conf.d/00-docker
RUN echo 'APT::Install-Recommends "0";' >> /etc/apt/apt.conf.d/00-docker

# Update package lists and install necessary build tools and common dependencies.
# These are essential for compiling some Python packages and for general test execution.
# Add any other system-level dependencies your specific Ansible collections might require.
# For example, if your collections interact with AWS, Azure, databases, etc., you might need:
# apt-get install -y --no-install-recommends libssl-dev libffi-dev openssh-client

## 1. Core System Utilities & Locales
# These rarely change and provide the base for everything else
RUN apt-get update -qqy && \
    apt-get -qqy --no-install-recommends install \
      apt-transport-https apt-utils locales ca-certificates \
      software-properties-common bash tar zip unzip gnupg \
      netbase tzdata curl wget git git-lfs tree less nano jq rsync && \
    sed -i '/en_US.UTF-8/s/^# //g' /etc/locale.gen && \
    locale-gen && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

## 2. Networking & Debugging Tools
RUN apt-get update -qqy && \
    apt-get -qqy --no-install-recommends install \
      dnsutils iproute2 openssh-client iputils-ping \
      inetutils-telnet ssh-askpass sshpass ssh-client shellcheck \
      groff rlwrap && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

## 3. Build Tools & Development Libraries
# Essential for compiling Python C-extensions or local builds
RUN apt-get update -qqy && \
    apt-get -qqy --no-install-recommends install \
      build-essential patch genisoimage dirmngr openssl \
      zlib1g-dev libncurses5-dev libgdbm-dev libnss3-dev \
      libpq-dev libmariadb-dev libsasl2-dev libldap2-dev \
      libssl-dev libsqlite3-dev libreadline-dev libffi-dev \
      libbz2-dev uuid-dev default-mysql-client && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

# Generate and set the locale
RUN sed -i 's/^# *en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/' /etc/locale.gen \
    && locale-gen

## Ensure the following are set after running `locale-gen`
ENV LANGUAGE=en_US:en
ENV LANG=en_US.UTF-8
ENV LC_ALL=en_US.UTF-8

## 4. Docker CLI (DinD Support)
# Kept separate so the Docker version can be updated independently
# --- Start Docker-in-Docker (DinD) support installation ---
# Install Docker CLI to enable running docker commands inside this container.
# This is crucial for scenarios like 'ansible-test units --docker' which
# require interacting with a Docker daemon.
# Modern Docker Repo Setup
RUN install -m 0755 -d /etc/apt/keyrings && \
    curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg && \
    chmod a+r /etc/apt/keyrings/docker.gpg && \
    echo \
        "deb [arch=\"$(dpkg --print-architecture)\" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian \
        $(. /etc/os-release && echo \"$VERSION_CODENAME\") stable" | \
        tee /etc/apt/sources.list.d/docker.list > /dev/null && \
    apt-get update && \
    apt-get install -y --no-install-recommends \
      docker-ce-cli \
      docker-compose-plugin \
      docker-buildx-plugin && \
    apt-get clean && rm -rf /var/lib/apt/lists/*
# --- End Docker-in-Docker (DinD) support installation ---

# 5. External Tools: Copy in from builder stage ---
# Copy the already-extracted binaries from the builder stage
COPY --from=builder /out/usr/local/bin/ /usr/local/bin/

#==========
# PIP - Install Python dependencies
#==========
## ref: https://github.com/pypa/pip/issues/11179#issuecomment-1152766374
ENV PIP_ROOT_USER_ACTION=ignore

RUN python -m pip install --upgrade pip

# Set the working user for the rest of the build and runtime to 'root'
# as your desired UID is 0.
USER root

RUN echo "Install ansible-core with ANSIBLE_CORE_VERSION: ${ANSIBLE_CORE_VERSION}"

# Install ansible-core and ansible-test.
# ansible-test is included as part of the ansible-core package.
# Use the specified ANSIBLE_CORE_VERSION.
#RUN pip install --user --upgrade pip \
RUN pip install --disable-pip-version-check --upgrade pip \
    && case "${ANSIBLE_CORE_VERSION}" in \
        latest) \
            pip install "ansible-core"; \
            ;; \
        devel) \
            # For 'devel', you might need to pull directly from Ansible's Git repo.
            # For simplicity, we'll treat 'devel' as 'latest' for this basic Dockerfile,
            # installing the latest published ansible-core.
#            set -x; \
            python -m pip install " https://github.com/ansible/ansible/archive/devel.tar.gz" --disable-pip-version-check --user; \
#            pip install "ansible-core"; \
#            set +x; \
            ;; \
        *) \
            # Use compatible release operator to get the latest patch version for a given minor version (e.g., 2.14.x)
            # Ensure the variable is not empty before attempting the install
            if [ -z "${ANSIBLE_CORE_VERSION}" ]; then \
                echo "Error: ANSIBLE_CORE_VERSION is empty for compatible release installation."; \
                exit 1; \
            fi; \
#            set -x; \
#            python -m pip install " https://github.com/ansible/ansible/archive/stable-${ANSIBLE_CORE_VERSION}.tar.gz" --disable-pip-version-check --user; \
#            pip install "ansible-core~=${ANSIBLE_CORE_VERSION}"; \
            MAJOR_MINOR=$(echo "${ANSIBLE_CORE_VERSION}" | cut -d. -f1,2); \
            NEXT_MINOR=$(echo "${ANSIBLE_CORE_VERSION}" | awk -F. '{print $2 + 1}'); \
            UPPER_BOUND="${MAJOR_MINOR%.*}.${NEXT_MINOR}"; \
            pip install --disable-pip-version-check "ansible-core>=${ANSIBLE_CORE_VERSION},<${UPPER_BOUND}"; \
#            set +x; \
            ;; \
    esac \
    && pip install --disable-pip-version-check pytest flake8
#    && pip install --user pytest flake8

COPY requirements.txt .
COPY requirements.yml .
COPY files/vsphere_automation_sdk-8.0.3.0.tar.gz /var/tmp/vsphere_automation_sdk.tar.gz

#RUN pip install --user -r requirements.txt
RUN pip install --disable-pip-version-check -r requirements.txt

## 2. Extract and install components manually from the local source
#RUN mkdir -p /tmp/sdk-build && \
#    tar -xzf /var/tmp/vsphere_automation_sdk.tar.gz -C /tmp/sdk-build --strip-components=1 && \
#    cd /tmp/sdk-build && \
#    # Install the specific internal requirements first
#    pip install --no-cache-dir -r requirements.txt && \
#    # Install the SDK itself in editable mode or direct install
#    pip install --no-cache-dir . && \
#    cd / && \
#    rm -rf /tmp/sdk-build

RUN mkdir -p /tmp/sdk-build && \
    tar -xzf /var/tmp/vsphere_automation_sdk.tar.gz -C /tmp/sdk-build --strip-components=1 && \
    # Force an older setuptools version that includes pkg_resources
    pip install --no-cache-dir "setuptools<70.0.0" && \
    pip install --no-cache-dir lxml>=4.3.0 six>=1.12 requests>=2.27.1 pyOpenSSL>=22.0.0 cryptography>=37.0.2 && \
    pip install --no-cache-dir \
        $(find /tmp/sdk-build/lib/vmwarecloud-aws -name "*.whl") \
        $(find /tmp/sdk-build/lib/vmwarecloud-draas -name "*.whl") \
        $(find /tmp/sdk-build/lib/nsx-python-sdk -name "*.whl") \
        $(find /tmp/sdk-build/lib/nsx-policy-python-sdk -name "*.whl") \
        $(find /tmp/sdk-build/lib/nsx-vmc-aws-integration-python-sdk -name "*.whl") \
        $(find /tmp/sdk-build/lib/nsx-vmc-policy-python-sdk -name "*.whl") && \
    cd /tmp/sdk-build && \
    pip install --no-cache-dir . && \
    cd / && \
    rm -rf /tmp/sdk-build

# Create the default Ansible collections directory to ensure
# `ansible-galaxy collection list` can run without errors.
# The `ansible-test` tool and other Ansible commands expect this path to exist.
RUN mkdir -p /root/.ansible/collections

#RUN ansible-galaxy collection install -r requirements.yml
## occasionally network related issues when running ansible-galaxy directly - so wrap with retry mechanism
# optionally can set environment variables to customize retries and delay
# ENV RETRY_MAX_ATTEMPTS=7
# ENV RETRY_SLEEP_SECONDS=15
RUN /usr/local/bin/retry_command.sh ansible-galaxy collection install -r requirements.yml

# Create a working directory for Ansible project files
WORKDIR /ansible_project
