```
-- скачиваем исходники
sudo apt -y install git mc && cd /tmp && git clone https://github.com/greenplum-db/gpdb.git && git submodule update --init

-- устанавливаем пакеты зависимостей
sudo add-apt-repository ppa:ubuntu-toolchain-r/test -y
sudo apt-get update
sudo apt-get install -y gcc-7 g++-7
sudo DEBIAN_FRONTEND=noninteractive  apt-get install -y \
	bison \
	ccache \
	cmake \
	curl \
	flex \
	git-core \
	gcc \
	g++ \
	krb5-kdc \
	krb5-admin-server \
        libkrb5-dev \
	inetutils-ping \
	libapr1-dev \
	libbz2-dev \
	libcurl4-gnutls-dev \
	libevent-dev \
	libpam-dev \
	libperl-dev \
	libreadline-dev \
	libssl-dev \
	libxerces-c-dev \
	libxml2-dev \
	libyaml-dev \
	libzstd-dev \
	locales \
	net-tools \
	ninja-build \
	openssh-client \
	openssh-server \
	openssl \
	pkg-config \
	python3-dev \
	python3-pip \
	python3-psycopg2 \
	python3-psutil \
	python3-yaml \
	zlib1g-dev

список смотрим тут
/tmp/gpdb/README.Ubuntu.bash





--собираем greenplum
cd /tmp/gpdb 
sudo PYTHON=/usr/bin/python3 ./configure --with-perl --with-python --with-libxml --prefix=/usr/local/gpdb '
sudo PYTHON=/usr/bin/python3 make -j4
sudo PYTHON=/usr/bin/python3 sudo make install
```
