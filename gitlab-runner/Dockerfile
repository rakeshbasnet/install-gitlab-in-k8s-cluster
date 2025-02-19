# Use the official GitLab Runner image as base
FROM gitlab/gitlab-runner:latest

# Install Docker (required for Docker-in-Docker)
USER root

# Install required dependencies and update packages
RUN apt-get update && apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg2 \
    lsb-release \
    software-properties-common \
    git \
    docker.io \
    && rm -rf /var/lib/apt/lists/*

# Install Docker and setup permissions (skip modifying docker.sock during build)
RUN getent group docker || groupadd docker && \
    usermod -aG docker root

# Set the default entrypoint to run GitLab Runner
ENTRYPOINT ["gitlab-runner" ,"run"]
