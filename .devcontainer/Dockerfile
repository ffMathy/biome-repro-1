FROM node:lts AS npm-packages
WORKDIR /src


FROM pulumi/pulumi-base:3.124.0 AS pulumi


FROM node:lts AS kubectl
WORKDIR /usr/local/bin

# install kubectl: https://stackoverflow.com/questions/51717471/how-to-install-kubectl-in-kubernetes-container-through-docker-image
RUN curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl" \
    && chmod +x ./kubectl \
    && curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 \
    && chmod +x get_helm.sh && ./get_helm.sh


FROM node:lts AS dev-container
WORKDIR /src
SHELL ["/bin/bash", "-c"]

# install global NPM packages
RUN npm install --force --global pnpm
RUN SHELL=bash pnpm setup
ENV PNPM_HOME="/root/.local/share/pnpm"
ENV PATH="/root/.local/share/pnpm:$PATH"
RUN pnpm install --force --global \
  nx \
  playwright \
  biome \
  vitest

RUN rm -f /etc/apt/apt.conf.d/docker-clean; echo 'Binary::apt::APT::Keep-Downloaded-Packages "true";' > /etc/apt/apt.conf.d/keep-cache
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
  --mount=type=cache,target=/var/lib/apt,sharing=locked \
  apt update && \
  playwright install-deps && \
  playwright install && \
  apt-get install --no-install-recommends -y \
  less \
  # install root certificates
  ca-certificates \
  # install JQ, needed for SSO integration for AWS: https://stackoverflow.com/questions/71789124/unable-to-use-jq-inside-docker-container-even-after-installing-jq-as-part-of-the
  jq \
  # install Azure CLI, needed for SSO integration for AWS: https://stackoverflow.com/questions/62030499/azure-cli-in-a-docker-container
  curl \
  apt-transport-https \
  lsb-release gnupg \
  # install Vim (often used to modify EKS ConfigMap): https://stackoverflow.com/questions/31515863/how-to-run-vi-on-docker-container
  vim \
  # install xdg to support opening browsers from within the devcontainer, needed for AWS SSO.
  xdg-utils \
  # install Python (needed for installation of UV)
  python3 \
  python3-pip \
  python3-dev \
  python3-venv \
  # install GCC (needed for installation of UV)
  gcc \
  # install and groff unzip (needed for installation of AWS CLI)
  unzip \
  groff

# install AWS CLI
RUN curl "https://awscli.amazonaws.com/awscli-exe-linux-$(uname -m).zip" -o "awscliv2.zip" && \
    unzip awscliv2.zip && ./aws/install

# install uv (needed for installing from requirements.txt)
RUN python3 -m pip install uv --break-system-packages

# install Docker: https://stackoverflow.com/questions/44451859/how-to-install-docker-in-docker-container
RUN curl -fsSL https://get.docker.com | sh

# install GitHub CLI: https://github.com/cli/cli/blob/trunk/docs/install_linux.md
RUN curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg | dd of=/usr/share/keyrings/githubcli-archive-keyring.gpg \
  && chmod go+r /usr/share/keyrings/githubcli-archive-keyring.gpg \
  && echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" | tee /etc/apt/sources.list.d/github-cli.list > /dev/null \
  && apt update \
  && apt install gh -y

# copy Pulumi binaries
ENV PATH="${PATH}:/root/pulumi/bin"
COPY --chmod=755 --from=pulumi /pulumi/bin /root/pulumi/bin
RUN pulumi version

# copy kubectl binaries
ENV PATH="${PATH}:/root/bin/kubectl"
COPY --chmod=755 --from=kubectl /usr/local/bin /root/bin/kubectl
RUN kubectl version --client=true

# sleeping forever is required for DevContainers.
CMD [ "sleep", "infinity" ]