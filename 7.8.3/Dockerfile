# Ignition Version
ARG IGNITION_VERSION="7.8.3"

FROM ubuntu:latest AS downloader
LABEL maintainer "Mack McGhee <mack@mivTechnologies.com>"
ARG IGNITION_VERSION

# Install some prerequisite packages
RUN apt-get update && apt-get install -y wget unzip

# Ignition Downloader Parameters
ARG IGNITION_FULL_AMD64_DOWNLOAD_URL="https://files.inductiveautomation.com/release/ia/build7.8.3/20160610-1659/Ignition-linux-64-7.8.3.zip"
ARG IGNITION_FULL_AMD64_DOWNLOAD_SHA256="ff4f530e7e9624afea919710a71d37e97e1c0c0bc9cb59d04e498cd50d214c5b"
ARG IGNITION_AMD64_JRE_SUFFIX="nix"

# Default Build Edition - FULL, EDGE
ARG BUILD_EDITION="FULL"

# Retrieve Ignition Installer and Perform Ignition Installation
ENV INSTALLER_PATH /root
ENV INSTALLER_NAME "ignition-install.zip"
WORKDIR ${INSTALLER_PATH}

# Set to Bash Shell Execution instead of /bin/sh
SHELL [ "/bin/bash", "-c" ]

# Download Installation Zip File based on Detected Architecture
RUN set -exo pipefail; \
    dpkg_arch="$(dpkg --print-architecture | awk '{print toupper($0)}')"; \
    download_url_env="IGNITION_${BUILD_EDITION}_${dpkg_arch}_DOWNLOAD_URL"; \
    download_sha256_env="IGNITION_${BUILD_EDITION}_${dpkg_arch}_DOWNLOAD_SHA256"; \
    if [ -n "${!download_url_env}" ] && [ -n "${!download_sha256_env}" ]; then \
        wget -q --referer https://inductiveautomation.com/* -O "${INSTALLER_NAME}" "${!download_url_env}" && \
        if [[ ${BUILD_EDITION} != *"NIGHTLY"* ]]; then echo "${!download_sha256_env}" "${INSTALLER_NAME}" | sha256sum -c -; fi ; \
    else \
        echo "Architecture ${dpkg_arch} download targets not defined, aborting build"; \
        exit 1; \
    fi


# Extract Installation Zip File
RUN mkdir ignition && \
    unzip -q ${INSTALLER_NAME} -d ignition/ && \
    chmod +x ignition/gwcmd.sh ignition/ignition-gateway ignition/ignition.sh

# Change to Ignition folder
WORKDIR ${INSTALLER_PATH}/ignition

# Stage data, temp, logs and user-lib in var folders
RUN mkdir -p /var/lib/ignition && \
    mv data /var/lib/ignition/ && \
    mv user-lib /var/lib/ignition/ && \
    mv temp /var/lib/ignition/data && \
    mv logs /var/log/ignition && \
    ln -s /var/lib/ignition/data data && \
    ln -s /var/lib/ignition/user-lib user-lib && \
    ln -s /var/lib/ignition/data/temp temp && \
    ln -s /var/lib/ignition/data/temp /var/lib/ignition/temp && \
    ln -s /var/log/ignition logs

# RUNTIME IMAGE
FROM adoptopenjdk:8-jre-hotspot as final
LABEL maintainer "Mack McGhee <mack@mivTechnologies.com>"
ARG IGNITION_VERSION

# Install some prerequisite packages
RUN apt-get update && apt-get install -y curl gettext procps pwgen zip unzip sqlite3 fontconfig fonts-dejavu

# Setup Install Targets
ENV IGNITION_INSTALL_LOCATION="/usr/local/share/ignition"
ENV IGNITION_INSTALL_USERHOME="/home/ignition"

# Build Arguments for UID/GID
ARG IGNITION_UID
ENV IGNITION_UID ${IGNITION_UID:-999}
ARG IGNITION_GID
ENV IGNITION_GID ${IGNITION_GID:-999}

# Setup dedicated user, map file permissions, and set execution flags
RUN mkdir ${IGNITION_INSTALL_USERHOME} && \
    (getent group ${IGNITION_GID} > /dev/null 2>&1 || groupadd -r ignition -g ${IGNITION_GID}) && \
    (getent passwd ${IGNITION_UID} > /dev/null 2>&1 || useradd -r -d ${IGNITION_INSTALL_USERHOME} -u ${IGNITION_UID} -g ${IGNITION_GID} ignition) && \
    chown ${IGNITION_UID}:${IGNITION_GID} ${IGNITION_INSTALL_USERHOME}

# Copy Ignition Installation from Build Image
COPY --chown=${IGNITION_UID}:${IGNITION_GID} --from=downloader /root/ignition ${IGNITION_INSTALL_LOCATION}
COPY --chown=${IGNITION_UID}:${IGNITION_GID} --from=downloader /var/lib/ignition /var/lib/ignition
COPY --chown=${IGNITION_UID}:${IGNITION_GID} --from=downloader /var/log/ignition /var/log/ignition

# Declare Healthcheck
HEALTHCHECK --interval=10s --start-period=60s --timeout=3s \
    CMD curl --max-time 3 -f http://localhost:8088/main/StatusPing 2>&1 | grep RUNNING

# Setup Port Expose
EXPOSE 8088 8043 8000

# Launch Ignition
USER ${IGNITION_UID}
WORKDIR ${IGNITION_INSTALL_LOCATION}

# Update path to include embedded java install location
ENV PATH="${IGNITION_INSTALL_LOCATION}/lib/runtime/jre/bin:${PATH}"

# Copy in Entrypoint and helper scripts
COPY docker-entrypoint.sh /usr/local/bin/
COPY accept-gwnetwork.sh /usr/local/bin/

# Prepare Execution Settings
ENTRYPOINT [ "docker-entrypoint.sh" ]
CMD [ "./ignition-gateway" \
    , "data/ignition.conf" \
    , "wrapper.syslog.ident=Ignition-Gateway" \
    , "wrapper.pidfile=./Ignition-Gateway.pid" \
    , "wrapper.name=Ignition-Gateway" \
    , "wrapper.displayname=Ignition-Gateway" \
    , "wrapper.statusfile=./Ignition-Gateway.status" \
    , "wrapper.java.statusfile=./Ignition-Gateway.java.status" ]