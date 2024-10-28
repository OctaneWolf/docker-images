ARG DEBIAN_RELEASE=bookworm
FROM debian:"${DEBIAN_RELEASE}" as jre-build
SHELL ["/bin/bash", "-e", "-u", "-o", "pipefail", "-c"]
ARG TARGETPLATFORM
COPY adoptium-get-jdk-link.sh /usr/bin/local/adoptium-get-jdk-link.sh
COPY adoptium-install-jdk.sh /usr/bin/local/adoptium-install-jdk.sh
ARG JAVA_VERSION=17.0.12_7
RUN set -x; apt-get update && \
  apt-get install --no-install-recommends -y ca-certificates jq curl &&\
  /usr/bin/local/adoptium-install-jdk.sh
ENV PATH="/opt/jdk-${JAVA_VERSION}/bin:${PATH}"
RUN if [[ "${TARGETPLATFORM}" != "linux/arm/v7" ]]; then \
    case "$(jlink --version 2>&1)" in \
      "17."*) set -- "--compress=2" ;; \
      "21."*) set -- "--compress=zip-6" ;; \
      *) echo "ERROR: unmanaged jlink version pattern" && exit 1 ;; \
    esac; \
    jlink \
      --strip-java-debug-attributes \
      "$1" \
      --add-modules ALL-MODULE-PATH \
      --no-man-pages \
      --no-header-files \
      --output /javaruntime; \
  else \
    cp -r "/opt/jdk-${JAVA_VERSION}" /javaruntime; \
  fi
FROM debian:"${DEBIAN_RELEASE}" AS agent
ARG user=jenkins
ARG group=jenkins
ARG uid=1000
ARG gid=1000
RUN groupadd -g "${gid}" "${group}" && useradd -l -c "Jenkins user" -d /home/"${user}" -u "${uid}" -g "${gid}" -m "${user}" || echo "user ${user} already exists."
ARG AGENT_WORKDIR=/home/"${user}"/agent
ENV TZ=Etc/UTC
RUN apt-get update \
  && apt-get --yes --no-install-recommends install ca-certificates curl fontconfig git git-lfs less netbase openssh-client patch tzdata python3 python3-pip python3-venv python3-full \
  && apt-get clean && rm -rf /tmp/* /var/cache/* /var/lib/apt/lists/*
ARG VERSION=3273.v4cfe589b_fd83
ADD --chown="${user}":"${group}" "https://repo.jenkins-ci.org/public/org/jenkins-ci/main/remoting/${VERSION}/remoting-${VERSION}.jar" /usr/share/jenkins/agent.jar
RUN chmod 0644 /usr/share/jenkins/agent.jar \
  && ln -sf /usr/share/jenkins/agent.jar /usr/share/jenkins/slave.jar
ENV LANG C.UTF-8
ENV JAVA_HOME=/opt/java/openjdk
COPY --from=jre-build /javaruntime "$JAVA_HOME"
ENV PATH="${JAVA_HOME}/bin:${PATH}"
USER "${user}"
ENV AGENT_WORKDIR=${AGENT_WORKDIR}
RUN mkdir -p /home/"${user}"/.jenkins && mkdir -p "${AGENT_WORKDIR}"
VOLUME /home/"${user}"/.jenkins
VOLUME "${AGENT_WORKDIR}"
WORKDIR /home/"${user}"
ENV USER=${user}
FROM agent AS inbound-agent
ARG user=jenkins
USER root
COPY jenkins-agent /usr/local/bin/jenkins-agent
RUN chmod +x /usr/local/bin/jenkins-agent && ln -s /usr/local/bin/jenkins-agent /usr/local/bin/jenkins-slave
USER ${user}
ENTRYPOINT ["/usr/local/bin/jenkins-agent"]