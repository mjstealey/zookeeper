FROM krallin/centos-tini:centos7
MAINTAINER Michael J. Stealey <michael.j.stealey@gmail.com>

ARG ZOOKEEPER_VERSION=3.4.11

# Set correct environment variables.
ENV	HOME=/root \
    LANG=en_US.UTF-8 \
    LC_ALL=en_US.UTF-8

# java 8: https://github.com/binarybabel/docker-jdk/blob/master/src/centos.Dockerfile
ENV JAVA_DOWNLOAD=http://download.oracle.com/otn-pub/java/jdk/8u161-b12/2f38c3b165be4555a1fa6e98c45e0808/jdk-8u161-linux-x64.rpm
RUN cd /tmp \
    && curl -o jdk.rpm -jfksSLH "Cookie: oraclelicense=accept-securebackup-cookie" \
       "${JAVA_DOWNLOAD:-$(curl -s https://lv.binarybabel.org/catalog-api/java/jdk8.txt?p=downloads.rpm)}" \
    && rpm -Uvh jdk.rpm && rm jdk.rpm \
    && echo "export JAVA_HOME=/usr/java/default/" > /etc/profile.d/java_home.sh

# zookeeper
WORKDIR /opt/
ENV ZOO_USER=zookeeper \
    ZOO_CONF_DIR=/conf \
    ZOO_DATA_DIR=/data \
    ZOO_DATA_LOG_DIR=/datalog \
    ZOO_PORT=2181 \
    ZOO_TICK_TIME=2000 \
    ZOO_INIT_LIMIT=5 \
    ZOO_SYNC_LIMIT=2 \
    ZOO_MAX_CLIENT_CNXNS=60

# Add a user and make dirs
RUN set -ex \
    && adduser "$ZOO_USER" \
    && mkdir -p "$ZOO_DATA_LOG_DIR" "$ZOO_DATA_DIR" "$ZOO_CONF_DIR" \
    && chown "$ZOO_USER:$ZOO_USER" "$ZOO_DATA_LOG_DIR" "$ZOO_DATA_DIR" "$ZOO_CONF_DIR"
ARG DISTRO_NAME=zookeeper-${ZOOKEEPER_VERSION}
RUN curl -o ${DISTRO_NAME}.tar.gz "https://www.apache.org/dist/zookeeper/$DISTRO_NAME/$DISTRO_NAME.tar.gz" \
    && tar -xzf zookeeper-${ZOOKEEPER_VERSION}.tar.gz \
    && mv "$DISTRO_NAME" zookeeper \
    && mv zookeeper/conf/* "$ZOO_CONF_DIR" \
    && rm -rf "$DISTRO_NAME.tar.gz" \
    && yum clean all

WORKDIR /opt/zookeeper/
VOLUME ["$ZOO_DATA_DIR", "$ZOO_DATA_LOG_DIR"]

EXPOSE $ZOO_PORT 2888 3888

ENV PATH=$PATH:/$DISTRO_NAME/bin \
    ZOOCFGDIR=$ZOO_CONF_DIR \
    PATH=$PATH:/opt/zookeeper/bin

COPY docker-entrypoint.sh /
ENTRYPOINT ["/usr/local/bin/tini", "--", "/docker-entrypoint.sh"]
CMD ["zkServer.sh", "start-foreground"]
