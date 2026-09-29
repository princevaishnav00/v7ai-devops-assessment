FROM ubuntu:24.04


ENV  DEBIAN_FRONTEND=noninteractive

RUN apt-get update && \
    apt-get install -y \
        openssh-server \
        sudo \
        nginx \
        ufw \
        curl \
        git \
        iproute2 \
        iputils-ping \
        net-tools \
        procps && \
    rm -rf /var/lib/apt/lists/*

RUN mkdir -p /run/sshd

CMD ["/usr/sbin/sshd", "-D"]
