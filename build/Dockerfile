FROM debian:bookworm
ARG USER_ID GROUP_ID USERNAME NODE_VERSION DELTA_VERSION
ARG REBUILD_TIMESTAMP

RUN echo '#!/bin/sh\nexit 101' > /usr/sbin/policy-rc.d && chmod +x /usr/sbin/policy-rc.d

RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    export DEBIAN_FRONTEND=noninteractive && \
    rm -f /etc/apt/apt.conf.d/docker-clean && \
    echo 'Binary::apt::APT::Keep-Downloaded-Packages "true";' > /etc/apt/apt.conf.d/keep-cache && \
    apt-get update && \
        apt-get install -y --no-autoremove --no-install-recommends ca-certificates curl locales gnupg && \
    echo "en_US.UTF-8 UTF-8" > /etc/locale.gen && locale-gen en_US.UTF-8 &&\
        mkdir -p /usr/share/keyrings && \
        curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg -o /tmp/githubcli.gpg && \
        cat /tmp/githubcli.gpg | gpg --dearmor -o /usr/share/keyrings/githubcli-archive-keyring.gpg && \
        rm -f /tmp/githubcli.gpg && \
    chmod 644 /usr/share/keyrings/githubcli-archive-keyring.gpg && \
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg]" \
    "https://cli.github.com/packages stable main" | tee /etc/apt/sources.list.d/github-cli.list > /dev/null && apt-get update && \
        apt-get install -y --no-autoremove --no-install-recommends apt-utils wget zsh fzf ca-certificates sudo git tmux iptables ipset gh unzip jq \
        procps vim nano less iputils-ping traceroute dnsutils netcat-openbsd net-tools xdg-utils

ENV LANG=en_US.UTF-8 \
    LANGUAGE=en_US:en \
    LC_ALL=en_US.UTF-8

RUN groupadd -g $GROUP_ID $USERNAME || true && \
    useradd -m -u $USER_ID -g $GROUP_ID -s /bin/bash $USERNAME

RUN ARCH=$(dpkg --print-architecture) && \
    wget -q -O /tmp/git-delta_${DELTA_VERSION}_${ARCH}.deb \
        https://github.com/dandavison/delta/releases/download/${DELTA_VERSION}/git-delta_${DELTA_VERSION}_${ARCH}.deb && \
    dpkg -i /tmp/git-delta_${DELTA_VERSION}_${ARCH}.deb && \
    rm /tmp/git-delta_${DELTA_VERSION}_${ARCH}.deb

USER $USERNAME
WORKDIR /home/$USERNAME

RUN sh -c "$(wget -O- https://github.com/deluan/zsh-in-docker/releases/download/v1.2.0/zsh-in-docker.sh)" -- \
    -p git \
    -p fzf \
    -a "source /usr/share/doc/fzf/examples/key-bindings.zsh" \
    -a "source /usr/share/doc/fzf/examples/completion.zsh" \
    -a 'export HISTFILE="/home/$USERNAME/.cache/zsh_history"' \
    -a 'export HISTSIZE=10000' \
    -a 'export SAVEHIST=10000' \
    -a 'setopt HIST_IGNORE_DUPS' \
    -a 'setopt SHARE_HISTORY' \
    -a 'setopt HIST_FCNTL_LOCK' \
    -a 'setopt APPEND_HISTORY' \
    -a 'export NVM_DIR="$HOME/.nvm"' \
    -a '[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"' \
    -a '[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"' \
    -x

RUN curl -LsSf https://astral.sh/uv/install.sh | sh

RUN echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc && \
    echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc

# uv is installed - venv creation is handled by entrypoint in shared folder

RUN git config --global core.pager delta && \
    git config --global interactive.diffFilter "delta --color-only" && \
    git config --global delta.navigate true && \
    git config --global delta.light false && \
    git config --global delta.side-by-side true

ENV DEVCONTAINER=true

ENV SHELL=/bin/zsh

ENV NVM_DIR="/home/$USERNAME/.nvm"
RUN curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash

RUN bash -c "source $NVM_DIR/nvm.sh && \
    if [[ \"$NODE_VERSION\" == '--lts' ]]; then \
        nvm install --lts && \
        nvm alias default 'lts/*'; \
    else \
        nvm install $NODE_VERSION && \
        nvm alias default $NODE_VERSION; \
    fi && \
    nvm use default"

RUN echo 'export NVM_DIR="$HOME/.nvm"' >> ~/.bashrc && \
    echo '[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"' >> ~/.bashrc && \
    echo '[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"' >> ~/.bashrc

RUN bash -c "source $NVM_DIR/nvm.sh && \
    nvm use default && \
    npm install -g @anthropic-ai/claude-code"

RUN cat >> ~/.zshrc <<'EOF'

if [[ -n "$PS1" ]] && command -v stty >/dev/null; then
  function _update_size {
    local rows cols
    { stty size } 2>/dev/null | read rows cols
    ((rows)) && export LINES=$rows COLUMNS=$cols
  }
  TRAPWINCH() { _update_size }
  _update_size
fi
EOF

RUN echo "shopt -s checkwinsize" >> ~/.bashrc

RUN cat > ~/.tmux.conf <<'EOF'

set -g aggressive-resize on

set -g default-terminal "screen-256color"
set -ga terminal-overrides ",xterm-256color:Tc"

set -g mouse on
set -g history-limit 10000
EOF

USER root

# Add sudoers entry for claude user (following Anthropic's model)
RUN echo "claude ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/claude && \
    chmod 0440 /etc/sudoers.d/claude

RUN cat > /etc/profile.d/00-winsize.sh <<'EOF'
if [ -n "$PS1" ] && command -v stty >/dev/null 2>&1; then
  _update_size() {
    local sz
    sz=$(stty size 2>/dev/null) || return
    export LINES=${sz%% *}  COLUMNS=${sz##* }
  }
  trap _update_size WINCH
  _update_size
fi
EOF

# {{PROFILE_INSTALLATIONS}}

# {{LABELS}}

# Copy init-firewall script
COPY --chmod=755 init-firewall /home/$USERNAME/init-firewall
RUN chown $USER_ID:$GROUP_ID /home/$USERNAME/init-firewall

# Copy generate-tools-readme script and tools-report.sh library
COPY --chmod=755 generate-tools-readme /usr/local/bin/generate-tools-readme
RUN mkdir -p /usr/local/lib/claudebox
COPY tools-report.sh /usr/local/lib/claudebox/tools-report.sh

USER $USERNAME
RUN bash -c "source $NVM_DIR/nvm.sh && claude --version"

WORKDIR /workspace
USER root

# Use REBUILD_TIMESTAMP to bust cache when templates change
ARG REBUILD_TIMESTAMP
RUN echo "Rebuild timestamp: ${REBUILD_TIMESTAMP:-none}" > /dev/null

COPY --chown=$USERNAME docker-entrypoint.sh /usr/local/bin/docker-entrypoint
RUN sed -i "s|DOCKERUSER|$USERNAME|g" /usr/local/bin/docker-entrypoint && \
    sed -i "s|DOCKERUSER|$USERNAME|g" /home/$USERNAME/init-firewall && \
    chmod +x /usr/local/bin/docker-entrypoint

ENTRYPOINT ["/usr/local/bin/docker-entrypoint"]
