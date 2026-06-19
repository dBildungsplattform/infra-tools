ARG EE_BASE_IMAGE="quay.io/centos/centos:stream9"
ARG PYCMD="/usr/bin/python3.12"
ARG PYPKG="python312"
ARG PKGMGR_PRESERVE_CACHE=""
ARG ANSIBLE_GALAXY_CLI_COLLECTION_OPTS=""
ARG ANSIBLE_GALAXY_CLI_ROLE_OPTS=""
ARG ANSIBLE_INSTALL_REFS="ansible-core==2.18.6 ansible-runner==2.4.1"
ARG PKGMGR="/usr/bin/dnf"

# Base build stage
FROM $EE_BASE_IMAGE AS base
USER root
ENV PIP_BREAK_SYSTEM_PACKAGES=1
ARG EE_BASE_IMAGE
ARG PYCMD
ARG PYPKG
ARG PKGMGR_PRESERVE_CACHE
ARG ANSIBLE_GALAXY_CLI_COLLECTION_OPTS
ARG ANSIBLE_GALAXY_CLI_ROLE_OPTS
ARG ANSIBLE_INSTALL_REFS
ARG PKGMGR

COPY _build/scripts/ /output/scripts/
COPY _build/scripts/entrypoint /opt/builder/bin/entrypoint
RUN $PKGMGR install $PYPKG -y ; if [ -z $PKGMGR_PRESERVE_CACHE ]; then $PKGMGR clean all; fi
RUN /output/scripts/pip_install $PYCMD
RUN $PYCMD -m pip install --no-cache-dir $ANSIBLE_INSTALL_REFS

# Galaxy build stage
FROM base AS galaxy
ARG EE_BASE_IMAGE
ARG PYCMD
ARG PYPKG
ARG PKGMGR_PRESERVE_CACHE
ARG ANSIBLE_GALAXY_CLI_COLLECTION_OPTS
ARG ANSIBLE_GALAXY_CLI_ROLE_OPTS
ARG ANSIBLE_INSTALL_REFS
ARG PKGMGR

RUN /output/scripts/check_galaxy
COPY _build /build
WORKDIR /build

RUN mkdir -p /usr/share/ansible
RUN ansible-galaxy role install $ANSIBLE_GALAXY_CLI_ROLE_OPTS -r requirements.yml --roles-path "/usr/share/ansible/roles"
RUN ansible-galaxy collection install $ANSIBLE_GALAXY_CLI_COLLECTION_OPTS -r requirements.yml --collections-path "/usr/share/ansible/collections"

# Builder build stage
FROM base AS builder
ENV PIP_BREAK_SYSTEM_PACKAGES=1
WORKDIR /build
ARG EE_BASE_IMAGE
ARG PYCMD
ARG PYPKG
ARG PKGMGR_PRESERVE_CACHE
ARG ANSIBLE_GALAXY_CLI_COLLECTION_OPTS
ARG ANSIBLE_GALAXY_CLI_ROLE_OPTS
ARG ANSIBLE_INSTALL_REFS
ARG PKGMGR

RUN $PYCMD -m pip install --no-cache-dir bindep pyyaml packaging

COPY --from=galaxy /usr/share/ansible /usr/share/ansible

COPY _build/requirements.txt requirements.txt
COPY _build/bindep.txt bindep.txt
RUN $PYCMD /output/scripts/introspect.py introspect --user-pip=requirements.txt --user-bindep=bindep.txt --write-bindep=/tmp/src/bindep.txt --write-pip=/tmp/src/requirements.txt
RUN /output/scripts/assemble

RUN dnf install -y unzip && \
    dnf clean all

ARG HELM_VERSION=3.18.2
ARG TERRAFORM_VERSION=1.5.4
ARG KUBECTL_VERSION=1.33.7
ARG OP_VERSION=2.31.0
RUN curl -LO https://releases.hashicorp.com/terraform/$TERRAFORM_VERSION/terraform_${TERRAFORM_VERSION}_linux_amd64.zip && unzip terraform_${TERRAFORM_VERSION}_linux_amd64.zip && install -o root -g root -m 0755 terraform /usr/local/bin/terraform
RUN curl -LO https://dl.k8s.io/release/v$KUBECTL_VERSION/bin/linux/amd64/kubectl && install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
RUN curl -LO https://get.helm.sh/helm-v$HELM_VERSION-linux-amd64.tar.gz &&  tar -xf helm-v$HELM_VERSION-linux-amd64.tar.gz && install -o root -g root -m 0755 linux-amd64/helm /usr/local/bin/helm
RUN curl -LO https://cache.agilebits.com/dist/1P/op2/pkg/v$OP_VERSION/op_linux_amd64_v$OP_VERSION.zip && unzip op_linux_amd64_v$OP_VERSION.zip && install -o root -g root -m 0755 op /usr/local/bin/op

# Final build stage
FROM base AS final
ENV PIP_BREAK_SYSTEM_PACKAGES=1
ARG EE_BASE_IMAGE
ARG PYCMD
ARG PYPKG
ARG PKGMGR_PRESERVE_CACHE
ARG ANSIBLE_GALAXY_CLI_COLLECTION_OPTS
ARG ANSIBLE_GALAXY_CLI_ROLE_OPTS
ARG ANSIBLE_INSTALL_REFS
ARG PKGMGR

RUN pip3 install --no-cache-dir --upgrade pip setuptools
RUN dnf upgrade --refresh -y && \
    dnf install -y openssl git && \
    dnf clean all
RUN /output/scripts/check_ansible $PYCMD

COPY --from=galaxy /usr/share/ansible /usr/share/ansible

COPY --from=builder /usr/local/bin/terraform /usr/local/bin/terraform
COPY --from=builder /usr/local/bin/kubectl /usr/local/bin/kubectl
COPY --from=builder /usr/local/bin/helm /usr/local/bin/helm
COPY --from=builder /usr/local/bin/op /usr/local/bin/op

ARG HELM_DIFF_HASH=5873f8d94712f014dc2bb329acae63b8ffbf569b #3.15.10
RUN git clone --depth 1 --revision $HELM_DIFF_HASH https://github.com/databus23/helm-diff.git && helm plugin install helm-diff && rm -rf helm-diff

RUN /output/scripts/install-from-bindep
RUN mkdir -p /runner && chgrp 0 /runner && chmod -R ug+rwx /runner
WORKDIR /runner
RUN $PYCMD -m pip install --no-cache-dir 'dumb-init==1.2.5'
RUN update-alternatives --install /usr/bin/python3 python3 /usr/bin/python3.12 1
LABEL ansible-execution-environment=true
USER 1000
ENTRYPOINT ["/opt/builder/bin/entrypoint", "dumb-init"]
CMD ["bash"]
