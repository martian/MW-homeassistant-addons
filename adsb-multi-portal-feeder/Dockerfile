ARG BUILD_FROM=thomx/fr24feed-piaware:1.32.1

# Rebuild dump1090 with custom additions

FROM debian:bullseye as dump1090

ENV DUMP1090_VERSION RTL-SDR-bias-tee

# DUMP1090
RUN apt-get update && \
    apt-get install -y \
    sudo \
    git-core \
    build-essential \
    debhelper \
    librtlsdr-dev \
    pkg-config \
    libncurses5-dev \
    libbladerf-dev && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /tmp
RUN git clone -b ${DUMP1090_VERSION} --depth 1 https://github.com/martian/dump1090 && \
    cd dump1090 && \
    make CPUFEATURES=no

FROM $BUILD_FROM

ENV TZ=UTC

# add bashio (thx https://github.com/hassio-addons/addon-debian-base/blob/main/base/Dockerfile#L47)
ADD https://github.com/hassio-addons/bashio/archive/v0.15.0.tar.gz  /tmp/bashio.tar.gz
RUN apt-get update && apt-get install -y curl jq && mkdir /tmp/bashio \
    && tar zxvf \
        /tmp/bashio.tar.gz \
        --strip 1 -C /tmp/bashio \
    \
    && mv /tmp/bashio/lib /usr/lib/bashio \
    && ln -s /usr/lib/bashio/bashio /usr/bin/bashio

# add hassio sensors (thx https://github.com/plo53/adsb-hassio-sensors)
ARG version=1.1.2-1
ARG sha256sum=2b8e660c304448972779011c3f15e37ba659403508c632bb498bcb41aeb4f75b
ADD https://github.com/plo53/adsb-hassio-sensors/archive/refs/tags/${version}.tar.gz /tmp/
RUN echo "${sha256sum}  /tmp/${version}.tar.gz" | sha256sum --check
RUN tar xvfz /tmp/${version}.tar.gz adsb-hassio-sensors-${version}/{etc,usr} --strip-components=1 -C /

# add the options.json to environment variable magic
COPY export-env-from-config.sh /export-env-from-config.sh

# Replace the dump1090 binary with one that supports bias tee control.
COPY --from=dump1090 /tmp/dump1090/dump1090  /usr/lib/fr24/

RUN find /etc/s6-overlay/s6-rc.d/ -name script | xargs sed -i '1 a\source /export-env-from-config.sh'
RUN find /etc/s6-overlay/s6-rc.d/ -name run | xargs sed -i '1 a\source /export-env-from-config.sh'


# Add the nice banner

ADD https://raw.githubusercontent.com/hassio-addons/addon-debian-base/v5.1.0/base/rootfs/etc/cont-init.d/00-banner.sh /etc/s6-overlay/s6-rc.d/banner/
RUN mkdir -p /etc/s6-overlay/s6-rc.d/banner && \
    echo "oneshot" > /etc/s6-overlay/s6-rc.d/banner/type && \
    echo "/etc/s6-overlay/s6-rc.d/banner/script" > /etc/s6-overlay/s6-rc.d/banner/up && \
    touch /etc/s6-overlay/s6-rc.d/user/contents.d/banner && \
    mv /etc/s6-overlay/s6-rc.d/banner/00-banner.sh /etc/s6-overlay/s6-rc.d/banner/script && \
    chmod +x /etc/s6-overlay/s6-rc.d/banner/script && \
    sed -i '1 s/^.*$/#!\/command\/with-contenv bashio/' /etc/s6-overlay/s6-rc.d/banner/script



# enhance timeout, as sometimes the api calls are slow (rate limited?)
ENV S6_CMD_WAIT_FOR_SERVICES_MAXTIME=30000
