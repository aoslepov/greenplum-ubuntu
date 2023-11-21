Заказываем ВМ под координаторы и воркеры
```
for i in {1..2}; do
yc compute instance create \
  --name gpdb-mdw-0$i \
  --hostname gpdb-mdw-0$i \
  --create-boot-disk size=15G,type=network-hdd,image-folder-id=standard-images,image-family=ubuntu-2004-lts \
  --cores 4 \
  --memory 4G \
  --network-interface subnet-name=default-ru-central1-a,nat-ip-version=ipv4 \
  --zone ru-central1-a \
  --metadata-from-file ssh-keys=/home/aslepov/meta.txt
done


for i in {1..2}; do
yc compute instance create \
  --name gpdb-sdw-0$i \
  --hostname gpdb-sdw-0$i \
  --create-boot-disk size=15G,type=network-hdd,image-folder-id=standard-images,image-family=ubuntu-2004-lts \
  --cores 4 \
  --memory 4G \
  --network-interface subnet-name=default-ru-central1-a,nat-ip-version=ipv4 \
  --zone ru-central1-a \
  --metadata-from-file ssh-keys=/home/aslepov/meta.txt
done
```

Cтавим пакеты для сборки, конфиг системы и лимиты
```
for i in {'158.160.37.130','158.160.124.70','158.160.63.31','158.160.127.44'}; do
# ставим пакеты для зависимостей
ssh -o StrictHostKeyChecking=no ubuntu@$i '
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
	zlib1g-dev'

## sysctl.conf
ssh ubuntu@$i '
echo "
vm.overcommit_memory = 2
vm.overcommit_ratio = 95
net.ipv4.ip_local_port_range = 10000 65535 # See Port Settings
kernel.sem = 250 2048000 200 8192
kernel.sysrq = 1
kernel.core_uses_pid = 1
kernel.msgmnb = 65536
kernel.msgmax = 65536
kernel.msgmni = 2048
net.ipv4.tcp_syncookies = 1
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.tcp_max_syn_backlog = 4096
net.ipv4.conf.all.arp_filter = 1
net.ipv4.ipfrag_high_thresh = 41943040
net.ipv4.ipfrag_low_thresh = 31457280
net.ipv4.ipfrag_time = 60
net.core.netdev_max_backlog = 10000
net.core.rmem_max = 2097152
net.core.wmem_max = 2097152
vm.swappiness = 10
vm.zone_reclaim_mode = 0
vm.zone_reclaim_mode = 0
vm.dirty_expire_centisecs = 500
vm.dirty_writeback_centisecs = 100
vm.dirty_background_ratio = 0 # See System Memory
vm.dirty_ratio = 0" | sudo tee -a /etc/sysctl.conf && sudo sysctl -p'

# limits
ssh ubuntu@$i '
echo "* soft nofile 1048576
* hard nofile 1048576
* soft nproc 1048576
* hard nproc 1048576" | sudo tee /etc/security/limits.d/greenplun.conf '


done
```


Установка GreenPlum
```
for i in {'158.160.37.130','158.160.124.70','158.160.63.31','158.160.127.44'}; do

# скачиваем сборку под ubuntu-2004
ssh -o StrictHostKeyChecking=no ubuntu@$i ' sudo wget https://github.com/aoslepov/greenplum-ubuntu/releases/download/greenplum-7/gpdb-2004.tar.gz -O /usr/local/gpdb-2004.tar.gz; cd /usr/local/ &&  sudo tar -xzvf /usr/local/gpdb-2004.tar.gz'

# добавляем юзера
ssh ubuntu@$i 'sudo groupadd gpadmin; sudo useradd gpadmin -r -m -g gpadmin; sudo chsh -s /bin/bash gpadmin'

# добавляем конфиг для машин кластера (машины имеют именно такой нейминг, в хостах должно быть соответствие в хостах)
ssh ubuntu@$i ' echo "cdw
scdw
sdw1
sdw2" | sudo tee /usr/local/gpdb/hostfile'

ssh -o StrictHostKeyChecking=no ubuntu@$i 'echo "10.128.0.22 cdw
10.128.0.19 scdw
10.128.0.36 sdw1
10.128.0.14 sdw2" | sudo tee -a /etc/hosts'


# добавляем env
ssh ubuntu@$i 'echo "source /usr/local/gpdb/greenplum_path.sh" | sudo tee -a /home/gpadmin/.bashrc'

# скачиваем конфиг (на стендбай координаторе поменять имя MASTER_HOST)
ssh ubuntu@$i 'sudo wget https://github.com/aoslepov/greenplum-ubuntu/blob/main/install/gpdb-2004.tar.gz -O /usr/local/gpdb/gpinitsystem_config;sudo chown -R gpadmin:gpadmin /usr/local/gpdb'

#создаём каталоги и даём права на них
ssh ubuntu@$i 'sudo mkdir -p /data/{mirror,coordinator} ; sudo chown -R gpadmin:gpadmin /data; sudo chown -R gpadmin:gpadmin /usr/local/gpdb '

done
```

Генерим ключи на серверах
```
-- генерим ключи
for i in {'158.160.37.130','158.160.124.70','158.160.63.31','158.160.127.44'}; do
ssh -o StrictHostKeyChecking=no ubuntu@$i 'sudo su gpadmin -c "ssh-keygen"'
done


-- собираем ключи в локальный файл
echo -n '' > /tmp/gpkeys
for i in {'158.160.37.130','158.160.124.70','158.160.63.31','158.160.127.44'}; do
ssh -o StrictHostKeyChecking=no ubuntu@$i 'sudo cat /home/gpadmin/.ssh/id_rsa.pub' >> /tmp/gpkeys
done


-- раскидываем ключи по машинам
for i in {'158.160.37.130','158.160.124.70','158.160.63.31','158.160.127.44'}; do
scp /tmp/gpkeys ubuntu@$i:/tmp/gpkeys
ssh -o StrictHostKeyChecking=no ubuntu@$i 'sudo su gpadmin -c "cat /tmp/gpkeys | tee /home/gpadmin/.ssh/authorized_keys; chmod 600 /home/gpadmin/.ssh/authorized_keys "'
done
echo -n '' > /tmp/gpkeys


-- прописываем сегменты только на координаторах
for i in {'158.160.37.130','158.160.124.70'}; do
ssh ubuntu@$i 'echo "export COORDINATOR_DATA_DIRECTORY=/data/coordinator/gpseg-1" | sudo tee -a /home/gpadmin/.bashrc'
done

-- далее на каждой ноде необходимо проверить связь по ssh и добавить в know-hosts
sudo su gpadmin
ssh cdw; exit
ssh scdw; exit
ssh sdw1; exit
ssh sdw2; exit
```


Инициализация кластера
```
-- на CDW
-- проверяем ключи
sudo su gpadmin
cd /usr/local/gpdb

gpssh-exkeys -f hostfile

[STEP 1 of 5] create local ID and authorize on local host
  ... /home/gpadmin/.ssh/id_rsa file exists ... key generation skipped

[STEP 2 of 5] keyscan all hosts and update known_hosts file

[STEP 3 of 5] retrieving credentials from remote hosts
  ... send to mdw-01
  ... send to mdw-02
  ... send to sdw-01
  ... send to sdw-02

[STEP 4 of 5] determine common authentication file content

[STEP 5 of 5] copy authentication files to all remote hosts
  ... finished key exchange with mdw-01
  ... finished key exchange with mdw-02
  ... finished key exchange with sdw-01
  ... finished key exchange with sdw-02

[INFO] completed successfully

-- инициализируем кластер с стендбаем scdw

gpinitsystem -c gpinitsystem_config -h hostfile -s scdw --mirror-mode=spread
20231121:13:19:49:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:-Checking configuration parameters, please wait...
20231121:13:19:49:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:-Reading Greenplum configuration file gpinitsystem_config
20231121:13:19:49:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:-Locale has not been set in gpinitsystem_config, will set to default value
20231121:13:19:49:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:-No DATABASE_NAME set, will exit following template1 updates
20231121:13:19:49:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:-COORDINATOR_MAX_CONNECT not set, will set to default value 250
20231121:13:19:49:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:-Checking configuration parameters, Completed
20231121:13:19:49:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:-Commencing multi-home checks, please wait...
....
20231121:13:19:50:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:-Configuring build for standard array
20231121:13:19:50:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:-Sufficient hosts for spread mirroring request
20231121:13:19:50:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:-Commencing multi-home checks, Completed
20231121:13:19:50:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:-Building primary segment instance array, please wait...
....
20231121:13:19:52:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:-Building spread mirror array type , please wait...
....
20231121:13:19:53:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:-Checking Coordinator host
20231121:13:19:53:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:-Checking new segment hosts, please wait...
........
20231121:13:20:01:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:-Checking new segment hosts, Completed
20231121:13:20:01:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:-Greenplum Database Creation Parameters
20231121:13:20:01:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:---------------------------------------
20231121:13:20:01:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:-Coordinator Configuration
20231121:13:20:01:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:---------------------------------------
20231121:13:20:01:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:-Coordinator hostname       = gpdb-mdw-01
20231121:13:20:01:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:-Coordinator port           = 5432
20231121:13:20:01:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:-Coordinator instance dir   = /data/coordinator/gpseg-1
20231121:13:20:01:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:-Coordinator LOCALE         =
20231121:13:20:01:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:-Greenplum segment prefix   = gpseg
20231121:13:20:01:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:-Coordinator Database       =
20231121:13:20:01:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:-Coordinator connections    = 250
20231121:13:20:01:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:-Coordinator buffers        = 128000kB
20231121:13:20:01:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:-Segment connections        = 750
20231121:13:20:01:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:-Segment buffers            = 128000kB
20231121:13:20:01:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:-Encoding                   = UNICODE
20231121:13:20:01:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:-Postgres param file        = Off
20231121:13:20:01:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:-Initdb to be used          = /usr/local/gpdb/bin/initdb
20231121:13:20:01:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:-GP_LIBRARY_PATH is         = /usr/local/gpdb/lib
20231121:13:20:01:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:-HEAP_CHECKSUM is           = on
20231121:13:20:01:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:-HBA_HOSTNAMES is           = 0
20231121:13:20:01:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:-Ulimit check               = Passed
20231121:13:20:01:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:-Array host connect type    = Single hostname per node
20231121:13:20:01:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:-Coordinator IP address [1]      = ::1
20231121:13:20:01:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:-Coordinator IP address [2]      = 10.128.0.22
20231121:13:20:01:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:-Coordinator IP address [3]      = fe80::d20d:fdff:fec6:c85a
20231121:13:20:01:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:-Standby Coordinator             = gpdb-mdw-02
20231121:13:20:01:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:-Number of primary segments = 1
20231121:13:20:01:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:-Standby IP address         = ::1
20231121:13:20:01:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:-Standby IP address         = 10.128.0.19
20231121:13:20:01:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:-Standby IP address         = fe80::d20d:79ff:fefd:7328
20231121:13:20:01:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:-Total Database segments    = 4
20231121:13:20:01:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:-Trusted shell              = ssh
20231121:13:20:01:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:-Number segment hosts       = 4
20231121:13:20:01:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:-Mirror port base           = 7000
20231121:13:20:01:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:-Number of mirror segments  = 1
20231121:13:20:01:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:-Mirroring config           = ON
20231121:13:20:01:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:-Mirroring type             = Spread
20231121:13:20:01:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:----------------------------------------
20231121:13:20:01:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:-Greenplum Primary Segment Configuration
20231121:13:20:01:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:----------------------------------------
20231121:13:20:01:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:-gpdb-mdw-01 	6000 	gpdb-mdw-01 	/data/gpseg0 	2
20231121:13:20:01:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:-gpdb-mdw-02 	6000 	gpdb-mdw-02 	/data/gpseg1 	3
20231121:13:20:01:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:-gpdb-sdw-01 	6000 	gpdb-sdw-01 	/data/gpseg2 	4
20231121:13:20:01:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:-gpdb-sdw-02 	6000 	gpdb-sdw-02 	/data/gpseg3 	5
20231121:13:20:01:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:---------------------------------------
20231121:13:20:01:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:-Greenplum Mirror Segment Configuration
20231121:13:20:01:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:---------------------------------------
20231121:13:20:01:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:-gpdb-mdw-02 	7000 	gpdb-mdw-02 	/data/mirror/gpseg0 	6
20231121:13:20:01:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:-gpdb-sdw-01 	7000 	gpdb-sdw-01 	/data/mirror/gpseg1 	7
20231121:13:20:01:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:-gpdb-sdw-02 	7000 	gpdb-sdw-02 	/data/mirror/gpseg2 	8
20231121:13:20:01:065446 gpinitsystem:gpdb-mdw-01:gpadmin-[INFO]:-gpdb-mdw-01 	7000 	gpdb-mdw-01 	/data/mirror/gpseg3 	9

## статус кластера
-- смотрим статус кластера
 gpstate
20231121:14:15:17:087072 gpstate:gpdb-mdw-01:gpadmin-[INFO]:-Starting gpstate with args:
20231121:14:15:17:087072 gpstate:gpdb-mdw-01:gpadmin-[INFO]:-local Greenplum Version: 'postgres (Greenplum Database) 7.0.0-beta.0+ build dev'
20231121:14:15:17:087072 gpstate:gpdb-mdw-01:gpadmin-[INFO]:-coordinator Greenplum Version: 'PostgreSQL 12.12 (Greenplum Database 7.0.0-beta.0+ build dev) on x86_64-pc-linux-gnu, compiled by gcc (Ubuntu 9.4.0-1ubuntu1~20.04.2) 9.4.0, 64-bit compiled on Nov 21 2023 07:55:45 Bhuvnesh C.'
20231121:14:15:17:087072 gpstate:gpdb-mdw-01:gpadmin-[INFO]:-Obtaining Segment details from coordinator...
20231121:14:15:17:087072 gpstate:gpdb-mdw-01:gpadmin-[INFO]:-Gathering data from segments...
20231121:14:15:17:087072 gpstate:gpdb-mdw-01:gpadmin-[INFO]:-Greenplum instance status summary
20231121:14:15:17:087072 gpstate:gpdb-mdw-01:gpadmin-[INFO]:-----------------------------------------------------
20231121:14:15:17:087072 gpstate:gpdb-mdw-01:gpadmin-[INFO]:-   Coordinator instance                                      = Active
20231121:14:15:17:087072 gpstate:gpdb-mdw-01:gpadmin-[INFO]:-   Coordinator standby                                       = scdw
20231121:14:15:17:087072 gpstate:gpdb-mdw-01:gpadmin-[INFO]:-   Standby coordinator state                                 = Standby host passive
20231121:14:15:17:087072 gpstate:gpdb-mdw-01:gpadmin-[INFO]:-   Total segment instance count from metadata                = 8
20231121:14:15:17:087072 gpstate:gpdb-mdw-01:gpadmin-[INFO]:-----------------------------------------------------
20231121:14:15:17:087072 gpstate:gpdb-mdw-01:gpadmin-[INFO]:-   Primary Segment Status
20231121:14:15:17:087072 gpstate:gpdb-mdw-01:gpadmin-[INFO]:-----------------------------------------------------
20231121:14:15:17:087072 gpstate:gpdb-mdw-01:gpadmin-[INFO]:-   Total primary segments                                    = 4
20231121:14:15:17:087072 gpstate:gpdb-mdw-01:gpadmin-[INFO]:-   Total primary segment valid (at coordinator)              = 4
20231121:14:15:17:087072 gpstate:gpdb-mdw-01:gpadmin-[INFO]:-   Total primary segment failures (at coordinator)           = 0
20231121:14:15:17:087072 gpstate:gpdb-mdw-01:gpadmin-[INFO]:-   Total number of postmaster.pid files missing              = 0
20231121:14:15:17:087072 gpstate:gpdb-mdw-01:gpadmin-[INFO]:-   Total number of postmaster.pid files found                = 4
20231121:14:15:17:087072 gpstate:gpdb-mdw-01:gpadmin-[INFO]:-   Total number of postmaster.pid PIDs missing               = 0
20231121:14:15:17:087072 gpstate:gpdb-mdw-01:gpadmin-[INFO]:-   Total number of postmaster.pid PIDs found                 = 4
20231121:14:15:17:087072 gpstate:gpdb-mdw-01:gpadmin-[INFO]:-   Total number of /tmp lock files missing                   = 0
20231121:14:15:17:087072 gpstate:gpdb-mdw-01:gpadmin-[INFO]:-   Total number of /tmp lock files found                     = 4
20231121:14:15:17:087072 gpstate:gpdb-mdw-01:gpadmin-[INFO]:-   Total number postmaster processes missing                 = 0
20231121:14:15:17:087072 gpstate:gpdb-mdw-01:gpadmin-[INFO]:-   Total number postmaster processes found                   = 4
20231121:14:15:17:087072 gpstate:gpdb-mdw-01:gpadmin-[INFO]:-----------------------------------------------------
20231121:14:15:17:087072 gpstate:gpdb-mdw-01:gpadmin-[INFO]:-   Mirror Segment Status
20231121:14:15:17:087072 gpstate:gpdb-mdw-01:gpadmin-[INFO]:-----------------------------------------------------
20231121:14:15:17:087072 gpstate:gpdb-mdw-01:gpadmin-[INFO]:-   Total mirror segments                                     = 4
20231121:14:15:17:087072 gpstate:gpdb-mdw-01:gpadmin-[INFO]:-   Total mirror segment valid (at coordinator)               = 4
20231121:14:15:17:087072 gpstate:gpdb-mdw-01:gpadmin-[INFO]:-   Total mirror segment failures (at coordinator)            = 0
20231121:14:15:17:087072 gpstate:gpdb-mdw-01:gpadmin-[INFO]:-   Total number of postmaster.pid files missing              = 0
20231121:14:15:17:087072 gpstate:gpdb-mdw-01:gpadmin-[INFO]:-   Total number of postmaster.pid files found                = 4
20231121:14:15:17:087072 gpstate:gpdb-mdw-01:gpadmin-[INFO]:-   Total number of postmaster.pid PIDs missing               = 0
20231121:14:15:17:087072 gpstate:gpdb-mdw-01:gpadmin-[INFO]:-   Total number of postmaster.pid PIDs found                 = 4
20231121:14:15:17:087072 gpstate:gpdb-mdw-01:gpadmin-[INFO]:-   Total number of /tmp lock files missing                   = 0
20231121:14:15:17:087072 gpstate:gpdb-mdw-01:gpadmin-[INFO]:-   Total number of /tmp lock files found                     = 4
20231121:14:15:17:087072 gpstate:gpdb-mdw-01:gpadmin-[INFO]:-   Total number postmaster processes missing                 = 0
20231121:14:15:17:087072 gpstate:gpdb-mdw-01:gpadmin-[INFO]:-   Total number postmaster processes found                   = 4
20231121:14:15:17:087072 gpstate:gpdb-mdw-01:gpadmin-[INFO]:-   Total number mirror segments acting as primary segments   = 0
20231121:14:15:17:087072 gpstate:gpdb-mdw-01:gpadmin-[INFO]:-   Total number mirror segments acting as mirror segments    = 4
20231121:14:15:17:087072 gpstate:gpdb-mdw-01:gpadmin-[INFO]:-----------------------------------------------------
```
