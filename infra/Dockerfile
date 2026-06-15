FROM ubuntu:22.04

RUN apt-get update && \
    apt-get install -y openssh-server && \
    mkdir -p /run/sshd && \ 
    rm -rf /var/lib/apt/lists/*


RUN 

CMD ["/usr/sbin/sshd", "-D"]