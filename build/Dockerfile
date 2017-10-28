FROM ubuntu:16.04

MAINTAINER Uni Mock <unimock@gate-m.de>

ARG SOGO_VERSION=3.2.10

RUN apt-get update && apt-get install -qy --no-install-recommends \
      mysql-client \
      inetutils-ping \
      rsync \
      tzdata \
      gnustep-make \
      gnustep-base-common \
      libgnustep-base-dev \
      make \
      gobjc \
      libxml2-dev \
      libssl-dev \
      libldap2-dev \
      libmysqlclient-dev \
      libmemcached-dev \
      libcurl4-openssl-dev \
      wget \
      cron \
      supervisor \
   && apt-get clean \
   && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*   

# --- set timezone
ARG BUILD_TZ="Europe/Berlin"
RUN echo $BUILD_TZ > /etc/timezone && \
    rm /etc/localtime && \
    ln -snf /usr/share/zoneinfo/$BUILD_TZ /etc/localtime && \
    dpkg-reconfigure -f noninteractive tzdata 

ARG GOSU_VERSION="1.10"
RUN echo "Getting gosu: ${GOSU_VERSION} and its signature" && \
    dpkgArch="$(dpkg --print-architecture | awk -F- '{ print $NF }')" && \
    wget -q -O /usr/local/bin/gosu "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$dpkgArch" && \
    chmod +x /usr/local/bin/gosu && \
    gosu nobody true

RUN echo "build SOPE-${SOGO_VERSION} ..." \
   && wget -q -O /tmp/SOPE-${SOGO_VERSION}.tar.gz https://github.com/inverse-inc/sope/archive/SOPE-${SOGO_VERSION}.tar.gz \
   && tar -C  /tmp -xzf /tmp/SOPE-${SOGO_VERSION}.tar.gz \
   && mkdir -p /build && mv /tmp/sope-SOPE-${SOGO_VERSION}/* /build/. \
   && cd /build && ./configure --with-gnustep --enable-debug --disable-strip && make && make install \
   && rm -rf /tmp/* /build

# ugly unimock patch for change password in multidomain setup against postfixadmin mysql user_sogo view
RUN echo "build SOGo-${SOGO_VERSION} ..."  \
   && wget -q  -O /tmp/SOGo-${SOGO_VERSION}.tar.gz https://github.com/inverse-inc/sogo/archive/SOGo-${SOGO_VERSION}.tar.gz \
   && tar -C  /tmp -xzf /tmp/SOGo-${SOGO_VERSION}.tar.gz \
   && mkdir -p /build && mv /tmp/sogo-SOGo-${SOGO_VERSION}/* /build/. \
   && echo "!!!!!!!!! START ugly unimock patch in /build/SoObjects/SOGo/SQLSource.m !!!!!!!!!!!!!!" \
   && sed -i 's/WHERE c_uid = /WHERE c_name = /'  /build/SoObjects/SOGo/SQLSource.m \
   && echo "!!!!!!!!! END   ugly unimock patch in /build/SoObjects/SOGo/SQLSource.m !!!!!!!!!!!!!!" \
   && cd /build && ./configure --enable-debug --disable-strip && make && make install \
   && rm -rf /tmp/* /build

RUN echo "sogo post installation ..." \
   && echo "/usr/local/lib/sogo" > /etc/ld.so.conf.d/sogo.conf  \
   && ldconfig \
   && echo "create user sogo" \
   && groupadd --system sogo && useradd --system --gid sogo sogo \
   && echo "create directories and enforce permissions" \
   && install -o sogo -g sogo -m 755 -d /var/run/sogo  \
   && install -o sogo -g sogo -m 750 -d /var/spool/sogo  \
   && install -o sogo -g sogo -m 750 -d /var/log/sogo

COPY etc/sogo   /etc/sogo
RUN touch /etc/sogo/sieve.creds

RUN echo '* * * * *   sogo   /usr/local/sbin/sogo-ealarms-notify 2>/dev/null' > /etc/cron.d/sogo && \
    echo '* * * * *   sogo   /usr/local/sbin/sogo-tool expire-sessions {{SOGO_APP_SESSION_EXPIRE}}' >> /etc/cron.d/sogo   && \
    echo '0 0 * * *   sogo   /usr/local/sbin/sogo-tool update-autoreply -p /etc/sogo/sieve.creds'   >> /etc/cron.d/sogo

COPY nginx     /nginx
VOLUME /nginx


############################################################################################

VOLUME /usr/local/lib/GNUstep/SOGo/WebServerResources

EXPOSE 20000

# load env
RUN . /usr/share/GNUstep/Makefiles/GNUstep.sh
############################################################################################
#
# docker-extensions
#
RUN mkdir -p /usr/local/bin
COPY ./bin/*             /usr/local/bin/
RUN chmod a+x /usr/local/bin/*
############################################################################################
#
# supervisor
#
RUN echo ". /etc/profile" > /root/.bashrc
RUN echo "export PS1='\H:\w\\$ '" >> /etc/profile
RUN echo "alias ll='ls -alF'"     >> /etc/profile
RUN echo 'export TERM="xterm"'    >> /etc/profile
# establisch supervisord
ADD /supervisor /etc/supervisor
RUN ln -sf /etc/supervisor/supervisord.conf /etc/supervisord.conf
RUN mkdir -p /var/log/supervisor && chmod a+rw var/log/supervisor
# link /etc/init.d/ startup scripts to supervisor
RUN ( cd /etc/supervisor/services.d ; ls * | while read i; do mv /etc/init.d/$i /etc/init.d/$i-orig ;  ln -sf /etc/supervisor/super-init.sh /etc/init.d/$i ; done 2>/dev/null )
RUN ln -sf    /etc/supervisor/systemctl /bin/systemctl
RUN chmod a+x /etc/supervisor/* /etc/supervisor/*.d/*
COPY /supervisor/invoke-rc.d /usr/sbin/invoke-rc.d
COPY /supervisor/reload      /sbin/reload
RUN  chmod a+x /sbin/reload /usr/sbin/invoke-rc.d
EXPOSE 9001
############################################################################################
#
# create directory for service volume
#
RUN mkdir -p /service ; chmod a+rwx /service
############################################################################################
#
# startup script
#
ADD ./start.sh /start.sh
RUN chmod 755 /start.sh
CMD ["/start.sh"]

