FROM alpine:edge

LABEL repository.hub="alexmasterov/alpine-php:jit" \
      repository.url="https://github.com/AlexMasterov/dockerfiles" \
      maintainer="Alex Masterov <alex.masterow@gmail.com>"

ARG PHP_CONFIG=/etc/php

ARG REALPATH_TURBO_TAG=v2.0.0
ARG REDIS_TAG=3.1.6
ARG PHPIREDIS_TAG=v1.0.0
ARG MONGODB_VERSION=1.3.4
ARG PHPV8_VERSION=0.2.1
ARG GRAPHQL_BRANCH=master

ARG LIBBSON_VERSION=1.9.2
ARG LIBMONGOC_VERSION=1.9.2
# https://github.com/AlexMasterov/dockerfiles#alpine-libv8
ARG V8_SOURCE=https://www.dropbox.com/s/vc7o88ublxshmz2/alpine-libv8-6.5.146.tar.gz
ARG V8_DIR=/usr/local/v8

RUN set -x \
  && apk add --update \
    ssmtp \
    tini \
  && addgroup -g 82 -S www-data \
  && adduser -u 82 -S -D -h /var/cache/www-data -s /sbin/nologin -G www-data www-data

RUN set -x \
  && apk add --virtual .php-build-dependencies \
    autoconf \
    binutils \
    bison \
    bzip2-dev \
    curl \
    curl-dev \
    file \
    freetype-dev \
    g++ \
    gcc \
    git \
    icu-dev \
    jpeg-dev \
    libmcrypt-dev \
    libpng-dev \
    libtool \
    libwebp-dev \
    libxml2-dev \
    libxslt-dev \
    libzip-dev \
    make \
    pcre2-dev \
    postgresql-dev \
    re2c \
    readline-dev \
    sqlite-dev \
  && : "---------- Proper libpcre2 ----------" \
  && (cd /usr/lib; \
      ln -sf libpcre2-posix.a libpcre2.a; ln -sf libpcre2-posix.so libpcre2.so) \
  && : "---------- Proper iconv ----------" \
  && apk add --no-cache --virtual .iconv-runtime-dependencies \
    --repository https://dl-3.alpinelinux.org/alpine/edge/testing/ \
    gnu-libiconv-dev \
  && : "---------- Replace binary and headers ----------" \
  && (mv /usr/bin/gnu-iconv /usr/bin/iconv; \
      mv /usr/include/gnu-libiconv/*.h /usr/include; rm -rf /usr/include/gnu-libiconv) \
  && : "---------- Build flags ----------" \
  && export LDFLAGS="-Wl,-O2 -Wl,--hash-style=both -pie" \
  && export CFLAGS="-O2 -march=native -fstack-protector-strong -fpic -fpie" \
  && export CPPFLAGS=${CFLAGS} \
  && export MAKEFLAGS="-j $(expr $(getconf _NPROCESSORS_ONLN) \+ 1)" \
  && : "---------- PHP ----------" \
  && PHP_BRANCH="jit-dynasm" \
  && git clone -b ${PHP_BRANCH} --depth 1 https://github.com/zendtech/php-src.git /tmp/php-src \
  && cd /tmp/php-src \
  && : "---------- Build ----------" \
  && ./buildconf --force \
  && ./configure \
    --prefix=/usr \
    --sysconfdir=${PHP_CONFIG} \
    --with-config-file-path=${PHP_CONFIG} \
    --with-config-file-scan-dir=${PHP_CONFIG}/conf.d \
    --without-pear \
    --disable-cgi \
    --disable-debug \
    --disable-ipv6 \
    --disable-phpdbg \
    --disable-rpath \
    --disable-static \
    --enable-bcmath \
    --enable-calendar \
    --enable-dom \
    --enable-exif \
    --enable-fd-setsize=$(ulimit -n) \
    --enable-fpm \
      --with-fpm-group=www-data \
      --with-fpm-user=www-data \
    --enable-ftp \
    --enable-inline-optimization \
    --enable-intl \
    --enable-json \
    --enable-libxml \
      --with-libxml-dir=/usr \
    --enable-mbregex \
    --enable-mbstring \
    --enable-opcache \
      --enable-huge-code-pages \
      --enable-opcache-file \
    --enable-pcntl \
    --enable-phar \
    --enable-session \
    --enable-shmop \
    --enable-soap \
    --enable-sockets \
    --enable-xml \
    --enable-xmlreader \
    --enable-xmlwriter \
    --enable-zip \
      --with-pcre-dir=/usr \
    --with-bz2=/usr \
    --with-curl=/usr \
    --with-gd \
      --with-freetype-dir=/usr \
      --with-jpeg-dir=/usr \
      --with-png-dir=/usr \
      --with-webp-dir=/usr \
      --with-xpm-dir=no \
      --with-zlib-dir=/usr \
    --with-iconv=/usr \
    --with-mhash \
    --with-openssl=/usr \
      --with-system-ciphers \
    --with-pcre-regex \
      --with-pcre-jit \
    --with-pdo-mysql=mysqlnd \
    --with-pdo-pgsql \
    --with-pdo-sqlite \
    --with-pgsql \
    --with-readline=/usr \
    --with-xmlrpc \
    --with-xsl=/usr \
  && make \
  && make install \
  && runtimeDeps="$( \
		scanelf --needed --nobanner --recursive /usr/sbin/php-fpm \
			| awk '{ gsub(/,/, "\nso:", $2); print "so:" $2 }' \
			| sort -u \
			| xargs -r apk info --installed \
			| sort -u \
	)" \
  && apk add --virtual .php-runtime-dependencies \
    ${runtimeDeps} \
  && : "---------- https://bugs.php.net/bug.php?id=52312 ----------" \
  && git clone -o ${REALPATH_TURBO_TAG} --depth 1 https://github.com/Whissi/realpath_turbo.git /tmp/realpath_turbo \
  && cd /tmp/realpath_turbo \
  && phpize \
  && ./configure \
  && make \
  && make install \
  && : "---------- Redis ----------" \
  && git clone -o ${REDIS_TAG} --depth 1 https://github.com/phpredis/phpredis.git /tmp/redis \
  && cd /tmp/redis \
  && phpize \
  && ./configure \
  && make \
  && make install \
  && : "---------- Phpiredis ----------" \
  && : "---------- https://blog.remirepo.net/post/2016/11/13/Redis-from-PHP ----------" \
  && apk add --virtual .phpiredis-build-dependencies \
    hiredis-dev \
  && apk add --virtual .phpiredis-runtime-dependencies \
    hiredis \
  && git clone -o ${PHPIREDIS_TAG} --depth 1 https://github.com/nrk/phpiredis.git /tmp/phpiredis \
  && cd /tmp/phpiredis \
  && phpize \
  && ./configure \
  && make \
  && make install \
  && apk del .phpiredis-build-dependencies \
  && : "---------- MongoDB ----------" \
  && apk add --virtual .mongodb-build-dependencies \
    cmake \
  && : "---------- libbson ----------" \
  && LIBBSON_FILENAME="libbson-${LIBBSON_VERSION}" \
  && LIBBSON_SOURCE="https://github.com/mongodb/libbson/releases/download/${LIBBSON_VERSION}/${LIBBSON_FILENAME}.tar.gz" \
  && curl -fSL --connect-timeout 30 ${LIBBSON_SOURCE} | tar xz -C /tmp \
  && cd /tmp/${LIBBSON_FILENAME} \
  && cmake \
    -DCMAKE_BUILD_TYPE=Release \
    -DENABLE_STATIC=OFF \
    -DENABLE_TESTS:BOOL=OFF \
    -DENABLE_EXAMPLES:BOOL=OFF \
    . \
  && make \
  && make install \
  && : "---------- mongo-c-driver ----------" \
  && LIBMONGOC_FILENAME="mongo-c-driver-${LIBMONGOC_VERSION}" \
  && LIBMONGOC_SOURCE="https://github.com/mongodb/mongo-c-driver/releases/download/${LIBMONGOC_VERSION}/${LIBMONGOC_FILENAME}.tar.gz" \
  && curl -fSL --connect-timeout 30 ${LIBMONGOC_SOURCE} | tar xz -C /tmp \
  && cd /tmp/${LIBMONGOC_FILENAME} \
  && cmake \
    -DCMAKE_BUILD_TYPE=Release \
    -DENABLE_AUTOMATIC_INIT_AND_CLEANUP:BOOL=OFF \
    -DENABLE_STATIC=OFF \
    -DENABLE_TESTS:BOOL=OFF \
    -DENABLE_EXAMPLES:BOOL=OFF \
    -DENABLE_SSL=LIBRESSL \
    -DENABLE_SASL=OFF \
    . \
  && make \
  && make install \
  && : "---------- mongo-php-driver ----------" \
  && apk add --virtual .mongodb-build-dependencies \
    pkgconfig \
  && apk add --virtual .mongodb-runtime-dependencies \
    libressl2.6-libtls \
  && MONGODB_FILENAME="mongodb-${MONGODB_VERSION}" \
  && MONGODB_SOURCE="https://github.com/mongodb/mongo-php-driver/releases/download/${MONGODB_VERSION}/${MONGODB_FILENAME}.tgz" \
  && curl -fSL --connect-timeout 30 ${MONGODB_SOURCE} | tar xz -C /tmp \
  && cd /tmp/${MONGODB_FILENAME} \
  && phpize \
  && ./configure \
    --with-libbson=system \
    --with-libmongoc=system \
  && make \
  && make install \
  && apk del .mongodb-build-dependencies \
  && : "---------- libv8 ----------" \
  && (mkdir -p ${V8_DIR}; \
      curl -fSL --connect-timeout 30 ${V8_SOURCE} | tar xz -C ${V8_DIR}) \
  && : "---------- php-v8 ----------" \
  && PHPV8_FILENAME="php-v8-${PHPV8_VERSION}" \
  && PHPV8_SOURCE="https://github.com/pinepain/php-v8/archive/v${PHPV8_VERSION}.tar.gz" \
  && curl -fSL --connect-timeout 30 ${PHPV8_SOURCE} | tar xz -C /tmp \
  && cd /tmp/${PHPV8_FILENAME} \
  && phpize \
  && ./configure \
    --with-v8=${V8_DIR} \
  && make \
  && make install \
  && : "---------- graphql-parser ----------" \
  && apk add --virtual .graphql-build-dependencies \
    bison \
    cmake \
    flex \
    python2 \
  && git clone -b ${GRAPHQL_BRANCH} --depth 1 https://github.com/dosten/graphql-parser-php.git /tmp/graphql \
  && : "---------- libgraphqlparser ----------" \
  && cd /tmp/graphql/deps/libgraphqlparser \
  && cmake . \
  && make \
  && make install \
  && : "---------- graphql-parser-php ----------" \
  && cd /tmp/graphql \
  && phpize \
  && ./configure \
  && make \
  && make install \
  && apk del .graphql-build-dependencies \
  && : "---------- Removing build dependencies, clean temporary files ----------" \
  && apk del .php-build-dependencies \
  && rm -rf /var/cache/apk/* /var/tmp/* /tmp/*

ENTRYPOINT ["tini", "--"]
CMD ["php-fpm"]
