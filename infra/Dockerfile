FROM ubuntu:22.04

RUN apt-get update && \
    apt-get install -y openssh-server && \
    mkdir -p /run/sshd && \
    ssh-keygen -A && \
    rm -rf /var/lib/apt/lists/*



CMD ["/usr/sbin/sshd", "-D"]