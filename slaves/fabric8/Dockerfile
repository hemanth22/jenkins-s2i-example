# The FROM will be replaced when building in OpenShift
FROM docker.io/fabric8/maven-builder

RUN curl -O https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm && \ 
    rpm -Uvh epel-release-latest-7*.rpm && \
    rm epel-release-latest-7*.rpm

RUN yum clean all && \
    export INSTALL_PKGS="nss_wrapper gettext" && \
    yum clean all && \
    yum install -y --setopt=tsflags=nodocs $INSTALL_PKGS && \
    rpm -V $INSTALL_PKGS && \
    yum clean all && \
    mkdir -p /var/lib/jenkins && \
    chown -R 1001:0 /var/lib/jenkins && \
    chmod -R g+w /var/lib/jenkins

# Copy the entrypoint
COPY configuration/* /var/lib/jenkins/

# Run the JNLP client by default
# To use swarm client, specify "/var/lib/jenkins/run-swarm-client" as Command
ENTRYPOINT ["/var/lib/jenkins/run-jnlp-client"]
