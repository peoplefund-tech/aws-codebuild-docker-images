FROM ubuntu:18.04

ENV PYTHON_VERSION="3.6.10" \
 DOCKER_VERSION="19.03.1" \
 DOCKER_COMPOSE_VERSION="1.24.0"

ARG CHINA_REGION

#****************        Utilities     *********************************************
ENV DOCKER_BUCKET="download.docker.com" \
    DOCKER_CHANNEL="stable" \
    DOCKER_SHA256="6e7d8e24ee46b13d7547d751696d01607d19c8224c1b2c867acc8c779e77734b" \
    DIND_COMMIT="3b5fac462d21ca164b3778647420016315289034" \
    GITVERSION_VERSION="4.0.0" \
    DEBIAN_FRONTEND="noninteractive" \
    SRC_DIR="/usr/src"


# Install git, SSH, and other utilities
RUN set -ex \
    && echo 'Acquire::CompressionTypes::Order:: "gz";' > /etc/apt/apt.conf.d/99use-gzip-compression \
    && apt-get update \
    && apt install -y apt-transport-https gnupg ca-certificates \
    && apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 3FA7E0328081BFF6A14DA29AA6A19B38D3D831EF \
    && apt-get update \
    && apt-get install software-properties-common -y --no-install-recommends \
    && apt-add-repository -y ppa:git-core/ppa \
    && apt-get update \
    && apt-get install git=1:2.* -y --no-install-recommends \
    && git version \
    && apt-get install -y --no-install-recommends openssh-client \
    && mkdir ~/.ssh \
    && touch ~/.ssh/known_hosts \
    && ssh-keyscan -t rsa,dsa -H github.com >> ~/.ssh/known_hosts \
    && ssh-keyscan -t rsa,dsa -H bitbucket.org >> ~/.ssh/known_hosts \
    && chmod 600 ~/.ssh/known_hosts \
    && apt-get install -y --no-install-recommends \
       wget python3 python3-dev python3-pip python3-setuptools fakeroot jq \
       netbase dirmngr bzr mercurial procps \
       tar gzip zip autoconf automake \
       bzip2 file g++ gcc imagemagick \
       libbz2-dev libc6-dev libcurl4-openssl-dev libdb-dev \
       libevent-dev libffi-dev libgeoip-dev libglib2.0-dev \
       libjpeg-dev libkrb5-dev liblzma-dev \
       libmagickcore-dev libmagickwand-dev libmysqlclient-dev \
       libncurses5-dev libpq-dev libreadline-dev \
       libsqlite3-dev libssl-dev libtool libwebp-dev \
       libxml2-dev libxslt1-dev libyaml-dev make \
       patch xz-utils zlib1g-dev unzip curl \
       e2fsprogs iptables xfsprogs \
       less groff liberror-perl \
       asciidoc build-essential bzr cvs cvsps docbook-xml docbook-xsl dpkg-dev \
       libdbd-sqlite3-perl libdbi-perl libdpkg-perl libhttp-date-perl \
       libio-pty-perl libserf-1-1 libsvn-perl libsvn1 libtcl8.6 libtimedate-perl \
       libxml2-utils libyaml-perl python-bzrlib python-configobj \
       sgml-base sgml-data subversion tcl tcl8.6 xml-core xmlto xsltproc \
       tk gettext gettext-base libapr1 libaprutil1 xvfb expect parallel \
       locales rsync net-tools mysql-client \
    && rm -rf /var/lib/apt/lists/* \
    && apt-get clean

# Install Docker
RUN set -ex \
    && curl -fSL "https://${DOCKER_BUCKET}/linux/static/${DOCKER_CHANNEL}/x86_64/docker-${DOCKER_VERSION}.tgz" -o docker.tgz \
    && echo "${DOCKER_SHA256} *docker.tgz" | sha256sum -c - \
    && tar --extract --file docker.tgz --strip-components 1  --directory /usr/local/bin/ \
    && rm docker.tgz \
    && docker -v \
# set up subuid/subgid so that "--userns-remap=default" works out-of-the-box
    && addgroup dockremap \
    && useradd -g dockremap dockremap \
    && echo 'dockremap:165536:65536' >> /etc/subuid \
    && echo 'dockremap:165536:65536' >> /etc/subgid \
    && wget "https://raw.githubusercontent.com/docker/docker/${DIND_COMMIT}/hack/dind" -O /usr/local/bin/dind \
    && curl -L https://github.com/docker/compose/releases/download/${DOCKER_COMPOSE_VERSION}/docker-compose-Linux-x86_64 > /usr/local/bin/docker-compose \
    && chmod +x /usr/local/bin/dind /usr/local/bin/docker-compose \
# Ensure docker-compose works
    && docker-compose version

# https://docs.aws.amazon.com/eks/latest/userguide/install-aws-iam-authenticator.html https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_CLI_installation.html
RUN curl -sS -o /usr/local/bin/aws-iam-authenticator https://amazon-eks.s3-us-west-2.amazonaws.com/1.11.5/2018-12-06/bin/linux/amd64/aws-iam-authenticator \
    && curl -sS -o /usr/local/bin/kubectl https://amazon-eks.s3-us-west-2.amazonaws.com/1.11.5/2018-12-06/bin/linux/amd64/kubectl \
    && curl -sS -o /usr/local/bin/ecs-cli https://s3.amazonaws.com/amazon-ecs-cli/ecs-cli-linux-amd64-latest \
    && chmod +x /usr/local/bin/kubectl /usr/local/bin/aws-iam-authenticator /usr/local/bin/ecs-cli

RUN set -ex \
    && pip3 install awscli boto3

VOLUME /var/lib/docker

# Configure SSH
COPY ssh_config /root/.ssh/config

COPY runtimes.yml /codebuild/image/config/runtimes.yml

COPY dockerd-entrypoint.sh /usr/local/bin/

#****************        PYTHON     *********************************************
ENV PATH="/usr/local/bin:$PATH" \
    GPG_KEY="0D96DF4D4110E5C43FBFB17F2D347EA6AA65421D" \
    PYTHON_PIP_VERSION="19.1.1" \
    LC_ALL=C.UTF-8 \
    LANG=C.UTF-8

RUN apt-get update && apt-get install -y --no-install-recommends \
        tcl-dev tk-dev \
    && rm -rf /var/lib/apt/lists/* \
    && wget -O python.tar.xz "https://www.python.org/ftp/python/${PYTHON_VERSION%%[a-z]*}/Python-$PYTHON_VERSION.tar.xz" && \
    wget -O python.tar.xz.asc "https://www.python.org/ftp/python/${PYTHON_VERSION%%[a-z]*}/Python-$PYTHON_VERSION.tar.xz.asc" \
    && export GNUPGHOME="$(mktemp -d)" \
    && (gpg --keyserver hkp://p80.pool.sks-keyservers.net:80 --recv-keys "$GPG_KEY" \
        || gpg --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys "$GPG_KEY" \
        || gpg --keyserver pgp.mit.edu --recv-keys "$GPG_KEY") \
    && gpg --batch --verify python.tar.xz.asc python.tar.xz \
    && rm -rf "$GNUPGHOME" python.tar.xz.asc \
    && mkdir -p /usr/src/python \
    && tar -xJC /usr/src/python --strip-components=1 -f python.tar.xz \
    && rm python.tar.xz \
    \
    && cd /usr/src/python \
    && ./configure \
        --enable-loadable-sqlite-extensions \
        --enable-shared \
    && make -j$(nproc) \
    && make install \
    && ldconfig \
# explicit path to "pip3" to ensure distribution-provided "pip3" cannot interfere
    && if [ ! -e /usr/local/bin/pip3 ]; then : \
        && wget -O /tmp/get-pip.py 'https://bootstrap.pypa.io/get-pip.py' \
        && python3 /tmp/get-pip.py "pip==$PYTHON_PIP_VERSION" \
        && rm /tmp/get-pip.py \
    ; fi \
# we use "--force-reinstall" for the case where the version of pip we're trying to install is the same as the version bundled with Python
# ("Requirement already up-to-date: pip==8.1.2 in /usr/local/lib/python3.6/site-packages")
# https://github.com/docker-library/python/pull/143#issuecomment-241032683
    && pip3 install --no-cache-dir --upgrade --force-reinstall "pip==$PYTHON_PIP_VERSION" \
        && pip install pipenv virtualenv --no-cache-dir \
        && pip3 install --no-cache-dir --upgrade setuptools wheel \
# then we use "pip list" to ensure we don't have more than one pip version installed
# https://github.com/docker-library/python/pull/100
    && [ "$(pip list |tac|tac| awk -F '[ ()]+' '$1 == "pip" { print $2; exit }')" = "$PYTHON_PIP_VERSION" ] \
    \
    && find /usr/local -depth \
        \( \
            \( -type d -a -name test -o -name tests \) \
            -o \
            \( -type f -a -name '*.pyc' -o -name '*.pyo' \) \
        \) -exec rm -rf '{}' + \
    && apt-get purge -y --auto-remove tcl-dev tk-dev \
    && rm -rf /usr/src/python ~/.cache \
    && cd /usr/local/bin \
    && { [ -e easy_install ] || ln -s easy_install-* easy_install; } \
    && ln -s idle3 idle \
    && ln -s pydoc3 pydoc \
    && ln -s python3 python \
    && ln -s python3-config python-config \
        && rm -fr /var/lib/apt/lists/* /tmp/* /var/tmp/*
#****************      END PYTHON     *********************************************

ENTRYPOINT ["dockerd-entrypoint.sh"]
