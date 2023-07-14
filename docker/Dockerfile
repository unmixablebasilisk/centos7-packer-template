FROM centos:7

ENV LANG en_US.UTF-8
ENV LC_ALL en_US.UTF-8

ARG FILE_SERVER_BASE={HASHICORP_PACKER_FILE_BASE}
ARG PACKER_VERSION=1.8.5
ARG PACKER_URL=$FILE_SERVER_BASE/packer/packer\_$PACKER_VERSION\_linux_amd64.zip

RUN set -x \
  && curl -fkSL "$PACKER_URL" -o ./packer\_$PACKER_VERSION\_linux_amd64.zip \
  && yum install -y deltarpm \
  && yum install -y epel-release \
  && yum install -y python36 python36-pip pbr unzip qemu-kvm libguestfs-tools python-passlib openssh-clients \
  && python3 -m pip install --upgrade "pip < 21.0" \
  && python3 -m pip install python-openstackclient ansible==2.10.7 python-openstackclient \
  && ansible-galaxy collection install community.general:==4.8.10 community.crypto ansible.posix \
  && unzip packer_${PACKER_VERSION}_linux_amd64.zip -d /usr/local/bin \
  && rm -f packer_${PACKER_VERSION}_linux_amd64.zip \
  && rm -f /etc/yum.repos.d/CentOS* \
  && yum clean all \
  && ln -s /usr/libexec/qemu-kvm /usr/libexec/qemu-system-x86_64
WORKDIR /mnt/packer
EXPOSE 5957
CMD ["/usr/local/bin/packer", "-version"]
