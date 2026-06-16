FROM ubuntu:22.04

ENV DEBIAN_FRONTEND=noninteractive

# Install SSH server and essential packages (including compilation tools)
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        openssh-server \
        sudo \
        curl \
        wget \
        gnupg2 \
        lsb-release \
        ca-certificates \
        iproute2 \
        net-tools \
        iputils-ping \
        procps \
        python3 \
        python3-apt \
        apt-transport-https \
        software-properties-common \
        build-essential \
        tcl \
        pkg-config \
        libssl-dev \
        tar && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Configure SSH
RUN mkdir -p /var/run/sshd && \
    sed -i 's/#PermitRootLogin.*/PermitRootLogin prohibit-password/' /etc/ssh/sshd_config && \
    sed -i 's/#PubkeyAuthentication.*/PubkeyAuthentication yes/' /etc/ssh/sshd_config && \
    sed -i 's/#PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config && \
    sed -i 's/PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config && \
    echo "StrictHostKeyChecking no" >> /etc/ssh/ssh_config && \
    echo "UserKnownHostsFile /dev/null" >> /etc/ssh/ssh_config

# Create ansible user with sudo privileges
RUN useradd -m -s /bin/bash ansible && \
    echo "ansible ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/ansible && \
    chmod 440 /etc/sudoers.d/ansible

# Setup SSH key directory for ansible user
RUN mkdir -p /home/ansible/.ssh && \
    chmod 700 /home/ansible/.ssh && \
    chown -R ansible:ansible /home/ansible/.ssh

# Setup SSH key directory for root
RUN mkdir -p /root/.ssh && \
    chmod 700 /root/.ssh

# Copy authorized keys (will be mounted or generated at build time)
COPY ssh_keys/id_rsa.pub /home/ansible/.ssh/authorized_keys
COPY ssh_keys/id_rsa.pub /root/.ssh/authorized_keys

RUN chmod 600 /home/ansible/.ssh/authorized_keys && \
    chown -R ansible:ansible /home/ansible/.ssh && \
    chmod 600 /root/.ssh/authorized_keys

EXPOSE 22

# Start SSH daemon in foreground
CMD ["/usr/sbin/sshd", "-D", "-e"]
