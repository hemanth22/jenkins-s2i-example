# The FROM will be replaced when building in OpenShift
FROM docker.io/fabric8/maven-builder

RUN mkdir -p /var/lib/jenkins && \
    chown -R 1001:0 /var/lib/jenkins && \
    chmod -R g+w /var/lib/jenkins && \
    mkdir -p /root/.m2

# Copy the entrypoint and Maven settings file
COPY configuration/* /var/lib/jenkins/
COPY m2/* /root/.m2/
#USER 1001

# Run the JNLP client by default
# To use swarm client, specify "/var/lib/jenkins/run-swarm-client" as Command
ENTRYPOINT ["/var/lib/jenkins/run-jnlp-client"]
