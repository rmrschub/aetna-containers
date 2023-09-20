FROM ubuntu:18.04

LABEL org.opencontainers.image.authors="René Schubotz"
LABEL org.opencontainers.image.source="https://github.com/rmrschub/aetna-containers/"

# Install useful linux packages
RUN apt update && \
    apt install -y \
        tar \
        xz-utils \
        tar && \
    rm -rf /var/lib/apt/lists

# s6-overlay
ARG S6_ARCH="x86_64"
ARG S6_OVERLAY_VERSION=3.1.2.1

ADD https://github.com/just-containers/s6-overlay/releases/download/v${S6_OVERLAY_VERSION}/s6-overlay-noarch.tar.xz /tmp
RUN tar -C / -Jxpf /tmp/s6-overlay-noarch.tar.xz
ADD https://github.com/just-containers/s6-overlay/releases/download/v${S6_OVERLAY_VERSION}/s6-overlay-${S6_ARCH}.tar.xz /tmp
RUN tar -C / -Jxpf /tmp/s6-overlay-${S6_ARCH}.tar.xz
ADD https://github.com/just-containers/s6-overlay/releases/download/v${S6_OVERLAY_VERSION}/s6-overlay-symlinks-noarch.tar.xz /tmp
RUN tar -C / -Jxpf /tmp/s6-overlay-symlinks-noarch.tar.xz
ADD https://github.com/just-containers/s6-overlay/releases/download/v${S6_OVERLAY_VERSION}/s6-overlay-symlinks-arch.tar.xz /tmp
RUN tar -C / -Jxpf /tmp/s6-overlay-symlinks-arch.tar.xz

ENTRYPOINT ["/init"]

# User setup
ARG NB_USER
ARG NB_GROUP
ARG NB_UID
ARG NB_PREFIX
ARG HOME

ENV NB_USER ${NB_USER:-jovyan}
ENV NB_GROUP ${NB_GROUP:-users}
ENV NB_UID ${NB_UID:-1000}
ENV NB_PREFIX ${NB_PREFIX:-/}
ENV S6_CMD_WAIT_FOR_SERVICES_MAXTIME 0
ENV HOME /home/$NB_USER
ENV SHELL /bin/bash

# Set shell to bash
SHELL ["/bin/bash", "-c"]

# Install useful linux packages
COPY packages.txt /tmp/packages.txt
RUN apt-get update && \
    apt-get install software-properties-common && \
    add-apt-repository ppa:deadsnakes/ppa && \
    sudo apt-get update && \
    apt-get install -yq $(cat /tmp/packages.txt) && \
    rm -rf /var/lib/apt/lists

# Install Github CLI
RUN type -p curl >/dev/null || (sudo apt update && sudo apt install curl -y); \
    curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg | sudo dd of=/usr/share/keyrings/githubcli-archive-keyring.gpg \
    && sudo chmod go+r /usr/share/keyrings/githubcli-archive-keyring.gpg \
    && echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" | sudo tee /etc/apt/sources.list.d/github-cli.list > /dev/null \
    && sudo apt update \
    && sudo apt install gh -y

# Create user and set required ownership
RUN useradd -M -s "$SHELL" -N -u ${NB_UID} ${NB_USER}; \
    if [[ -n "$HOME" && ! -d "$HOME" ]]; then \
        mkdir -p "${HOME}"; \
        chown "$NB_USER:$NB_GROUP" -R "$HOME"; \
    fi; \
    if [[ ! -f /etc/sudoers ]] || ! grep -q "^${NB_USER}[[:space:]]" /etc/sudoers; then \
        if [[ ! -f /etc/sudoers ]]; then \
            touch /etc/sudoers; \
        fi; \
        chmod 0660 /etc/sudoers; \
        echo "${NB_USER} ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers; \
        chmod 0440 /etc/sudoers; \
    fi;

# set locale configs
RUN echo "en_US.UTF-8 UTF-8" > /etc/locale.gen \
 && locale-gen
ENV LANG en_US.UTF-8
ENV LANGUAGE en_US.UTF-8
ENV LC_ALL en_US.UTF-8


# Install kubectl
ARG KUBECTL_ARCH="amd64"
ARG KUBECTL_VERSION=v1.21.0
ARG KUBECTL_INSTALL=1

RUN set -ex; \
    if [ "x${KUBECTL_INSTALL}" != "x" ]; then \
      curl -sL "https://dl.k8s.io/release/${KUBECTL_VERSION}/bin/linux/${KUBECTL_ARCH}/kubectl" -o /usr/local/bin/kubectl; \
      curl -sL "https://dl.k8s.io/${KUBECTL_VERSION}/bin/linux/${KUBECTL_ARCH}/kubectl.sha256" -o /tmp/kubectl.sha256; \
      echo "$(cat /tmp/kubectl.sha256) /usr/local/bin/kubectl" | sha256sum --check; \
      rm /tmp/kubectl.sha256; \
      chmod +x /usr/local/bin/kubectl; \
    fi;

# Install useful python packages
COPY requirements.txt /tmp/requirements.txt
RUN set -ex; \
    \
    python3 -m pip install -r /tmp/requirements.txt; 

# Install VS Code Server
ARG CODESERVER_VERSION=v4.7.1
RUN set -ex; \
    \
    apt-get update -yq; \
    curl -sL "https://github.com/cdr/code-server/releases/download/${CODESERVER_VERSION}/code-server_${CODESERVER_VERSION/v/}_amd64.deb" -o /tmp/code-server.deb; \
    dpkg -i /tmp/code-server.deb; \
    rm -f /tmp/code-server.deb; \
    \
    if code_server_path=$(command -v code-server 2>/dev/null); then \
        code_server_dir=$(dirname -- "$code_server_path"); \
        if [ ! -e "$code_server_dir/code" ]; then \
            printf "#!/usr/bin/env sh\nexec code-server \"\$@\"\n" > "$code_server_dir/code"; \
            chmod 0755 "$code_server_dir/code"; \
        fi; \
    fi; \
    \
    apt-get clean; \
    rm -rf /var/lib/apt/lists/*;

# Install VS Code Server extensions
COPY extensions.txt /tmp/extensions.txt
RUN set -ex; \
    \
    while IFS= read -r line; do \
        stripped_line="$(echo "$line" | sed -e 's/^[[:space:]]*//' -e 's/[[:space:]]*$//')"; \
        case "$stripped_line" in \
            '#'* | "") ;; \
            *) \
                code-server --install-extension "$stripped_line"; \
            ;; \
        esac; \
    done < /tmp/extensions.txt; \
    \
    code-server --list-extensions --show-versions;

# s6 - copy scripts
COPY --chown=root:root s6/ /etc
RUN chmod +x /etc/services.d/code-server/run

# s6 - 01-copy-tmp-home
RUN set -ex; \
    mkdir -p /tmp_home; \
    cp -r "${HOME}" /tmp_home; \
    chown -R "${NB_USER}:${NB_GROUP}" /tmp_home;


USER $NB_USER