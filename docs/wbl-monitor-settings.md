#########################################################################
# 1. 서버목록
- mon01.wbl.com
  public ip: 192.168.0.120
  private ip: 192.168.56.120
  관리자계정(sudo): wbl
- es01.wbl.com
  private ip: 192.168.56.121
  관리자계정(sudo): wbl
- es02.wbl.com
  private ip: 192.168.56.122
  관리자계정(sudo): wbl
- es03.wbl.com
  private ip: 192.168.56.123
  관리자계정(sudo): wbl

#########################################################################
# 2. 서버 공통 구성

## 2.1 기본패키지 설치
```
sudo yum install net-tools nfs-utils wget zip unzip tar  chrony -y
```

## 2.2 ntp 설정

### 2.2.1 mon01.wbl.com
- /etc/chrony.conf 작성
```
sudo tee /etc/chrony.conf<<EOF
server rocky.pool.ntp.org iburst

sourcedir /run/chrony-dhcp

driftfile /var/lib/chrony/drift

makestep 1.0 3

rtcsync

allow 192.168.0.0/16

keyfile /etc/chrony.keys

ntsdumpdir /var/lib/chrony

leapsectz right/UTC

logdir /var/log/chrony
EOF
```
- chrony 실행
```
sudo systemctl daemon-reload
sudo systemctl enable chronyd
sudo systemctl start chronyd
```

### 2.2.2 나머지 호스트

- es01.wbl.com es02.wbl.com es03.wbl.com 
```
sudo tee /etc/chrony.conf<<EOF
server 192.168.56.120 iburst

sourcedir /run/chrony-dhcp

driftfile /var/lib/chrony/drift

makestep 1.0 3

rtcsync

keyfile /etc/chrony.keys

ntsdumpdir /var/lib/chrony

leapsectz right/UTC

logdir /var/log/chrony
EOF
```
- chrony 실행
```
sudo systemctl daemon-reload
sudo systemctl enable chronyd
sudo systemctl start chronyd
```

## 2.3 firewalld & selinux 비활성화

### 2.3.1 firewalld 비활성화

```
sudo systemctl disable firewalld
sudo systemctl stop firewalld
```

### 2.3.2 selinux 비활성화

```
sudo setenforce 0
sudo vi /etc/sysconfig/selinux
#아래의 라인 수정 `SELINUX=enforcing => SELINUX=disabled` 저장
# reboot
```

## 2.4 ssh root login 임시허용

```
sudo vi /etc/ssh/sshd_config
# 아래의 라인 수정
# `PermitRootLogin prohibit-password => PermitRootLogin yes`
# Daemon restart
sudo systemctl daemon-reload
sudo systemctl restart sshd
#` 모든 설치 끝나면 PermitRootLogin prohibit-password로 원복`
```

## 2.5 호스트간 ssh 자동 접속설정

```
ssh-keygen -t rsa

ssh-copy-id mon01.wbl.com
ssh-copy-id es01.wbl.com
ssh-copy-id es02.wbl.com
ssh-copy-id es03.wbl.com
```

## 2.6 nfs-server 구성

### 2.6.1 환경구성
- 대상: mon01.wbl.com
```
sudo mkdir -p /data/nfsshared
sudo chown wbl:wbl /data
sudo chown nobody:nobody /data/nfsshared
sudo chmod 777 /data/nfsshared
```

### 2.6.2 exports 작성

```
sudo tee /etc/exports<<EOF
/data/nfsshared 192.168.0.0/16(rw,sync,no_subtree_check)
EOF
```

### 2.6.3 nfs-server 실행

```
sudo systemctl daemon-reload
sudo systemctl enable nfs-server
sudo systemctl start nfs-server
```

### 2.6.4 nfs volume 마운트

- 대상: es01.wbl.com es02.wbl.com es03.wbl.com
```
# 볼룸생성
sudo mkdir -p /data/nfsshared
sudo chown wbl:wbl /data
sudo chown nobody:nobody /data/nfsshared
sudo chmod 777 /data/nfsshared

# fstab 등록
# vi /etc/fstab
# 최하단에 아래 라인 추가
# 192.168.56.120:/data/nfsshared /data/nfsshared  nfs  defaults  0  0

# 볼륨마운트
sudo systemctl daemon-reload
sudo mount -a
```

#########################################################################
# 3 EFK 설치

#########################################################################
## 3.1 zookeeper 설치

### 3.1.1 다운로드 및 설치

```
# 패키지 다운로드
wget https://www.apache.org/dyn/closer.lua/zookeeper/zookeeper-3.8.5/apache-zookeeper-3.8.5-bin.tar.gz
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-9.2.1-linux-x86_64.tar.gz
tar zxf apache-zookeeper-3.8.5-bin.tar.gz
sudo mv apache-zookeeper-3.8.5 /opt/zookeeper
tar zxf elasticsearch-9.2.1-linux-x86_64.tar.gz
sudo mv elasticsearch-9.2.1 /opt/elasticsearch-9.2.1
sudo ln -sf /opt/elasticsearch-9.2.1 /opt/elasticsearch
sudo ln -sf /opt/elasticsearch/jdk /opt/elasticsearch/jdk

# Working 디렉토리 구성
sudo mkdir -p /data/zoo/conf
sudo mkdir -p /data/zoo/data
sudo mkdir -p /data/zoo/logs
sudo chown -R wbl:wbl /data

# 설정파일 복사
cp /opt/zookeeper/conf/* /data/zoo/conf/

# zoo.cfg 작성
tee /data/zoo/conf/zoo.cfg<<EOF
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/data/zoo/data
clientPort=2181
maxClientCnxns=60
#autopurge.snapRetainCount=3
#autopurge.purgeInterval=1
#metricsProvider.className=org.apache.zookeeper.metrics.prometheus.PrometheusMetricsProvider
#metricsProvider.httpHost=0.0.0.0
#metricsProvider.httpPort=7000
#metricsProvider.exportJvmInfo=true
server.1=192.168.56.121:2888:3888
server.2=192.168.56.122:2888:3888
server.3=192.168.56.123:2888:3888
EOF
```

### 3.1.2 myid 설정

- es01.wbl.com
```
tee /data/zoo/myid<<EOF
1
EOF
```
- es02.wbl.com
```
tee /data/zoo/myid<<EOF
2
EOF
```
- es03.wbl.com
```
tee /data/zoo/myid<<EOF
3
EOF
```

### 3.1.3 systemd 자동실행 등록

```
sudo tee /etc/systemd/system/zoo.service<<EOF
[Unit]
Description=Apache Zookeeper
After=network.target

[Service]
Type=forking
User=wbl
Group=wbl
SyslogIdentifier=zookeeper
Restart=on-failure
RestartSec=0s
Environment=ZK_SERVER_HEAP=128
Environment=JAVA_HOME=/opt/jdk
Environment=ZK_HOME=/opt/zookeeper
Environment=ZOOCFGDIR=/data/zoo/conf
Environment=ZOO_LOG_DIR=/data/zoo/logs
ExecStart=/opt/zookeeper/bin/zkServer.sh start /data/zoo/conf/zoo.cfg
ExecStop=/opt/zookeeper/bin/zkServer.sh stop /data/zoo/conf/zoo.cfg
ExecReload=/opt/zookeeper/bin/zkServer.sh restart /data/zoo/conf/zoo.cfg
WorkingDirectory=/data/zoo/data

[Install]
WantedBy=multi-user.target
EOF
```
실행
```
sudo systemctl daemon-reload
sudo systemctl enable zoo
sudo systemctl start zoo
```

#########################################################################
## 3.2 kafka 설치

### 3.2.1 다운로드 및 설치

```
# 패키지 다운로드
wget https://archive.apache.org/dist/kafka/3.8.1/kafka_2.13-3.8.1.tgz
tar zxf kafka_2.13-3.8.1.tgz
sudo mv kafka_2.13-3.8.1 /opt/kafka_2.13-3.8.1
sudo ln -sf /opt/kafka_2.13-3.8.1 /opt/kafka

# 환경구성
mkdir -p /data/kafka/data
mkdir -p /data/kafka/logs
```

### 3.2.2 server.properties 작성

- es01.wbl.com
```
tee /opt/kafka/config/server.properties<<EOF
broker.id=1
listeners=PLAINTEXT://192.168.56.121:9092

num.network.threads=3
num.io.threads=8

socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600
log.dirs=/data/kafka/logs

num.partitions=1
num.recovery.threads.per.data.dir=1

offsets.topic.replication.factor=1
transaction.state.log.replication.factor=1
transaction.state.log.min.isr=1
log.retention.hours=168
log.retention.check.interval.ms=300000

zookeeper.connect=192.168.56.121:2181,192.168.56.122:2181,192.168.56.123:2181
zookeeper.connection.timeout.ms=18000

group.initial.rebalance.delay.ms=0
EOF
```
- es02.wbl.com
```
tee /opt/kafka/config/server.properties<<EOF
broker.id=2
listeners=PLAINTEXT://192.168.56.122:9092

num.network.threads=3
num.io.threads=8

socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600
log.dirs=/data/kafka/logs

num.partitions=1
num.recovery.threads.per.data.dir=1

offsets.topic.replication.factor=1
transaction.state.log.replication.factor=1
transaction.state.log.min.isr=1
log.retention.hours=168
log.retention.check.interval.ms=300000

zookeeper.connect=192.168.56.121:2181,192.168.56.122:2181,192.168.56.123:2181
zookeeper.connection.timeout.ms=18000

group.initial.rebalance.delay.ms=0
EOF
```
- es03.wbl.com
```
tee /opt/kafka/config/server.properties<<EOF
broker.id=3
listeners=PLAINTEXT://192.168.56.123:9092

num.network.threads=3
num.io.threads=8

socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600
log.dirs=/data/kafka/logs

num.partitions=1
num.recovery.threads.per.data.dir=1

offsets.topic.replication.factor=1
transaction.state.log.replication.factor=1
transaction.state.log.min.isr=1
log.retention.hours=168
log.retention.check.interval.ms=300000

zookeeper.connect=192.168.56.121:2181,192.168.56.122:2181,192.168.56.123:2181
zookeeper.connection.timeout.ms=18000

group.initial.rebalance.delay.ms=0
EOF
```

### 3.2.3 systemd 자동 실행등록

```
sudo tee /etc/systemd/system/kafka.service<<EOF
[Unit]
Description=Apache Kafka
After=network.target

[Service]
Type=simple
User=wbl
Group=wbl
SyslogIdentifier=kafka
Restart=on-failure
Environment="KAFKA_HEAP_OPTS=-Xmx512M -Xms512M"
Environment=JAVA_HOME=/opt/jdk
Environment=KFK_HOME=/opt/kafka
Environment=LOG_DIR=/data/kafka/logs
ExecStart=/opt/kafka/bin/kafka-server-start.sh /opt/kafka/config/server.properties
ExecStop=/opt/kafka/bin/kafka-server-stop.sh /opt/kafka/config/server.properties

[Install]
WantedBy=multi-user.target
EOF
```
- 실행
```
sudo systemctl daemon-reload
sudo systemctl enable kafka
sudo systemctl start kafka
```

#########################################################################

## 3.2 Elasticsearch

### 3.2.1 환경구성

```
mkdir -p /data/elasticsearch/data
mkdir -p /data/elasticsearch/logs
```

### 3.2.2 jvm option 작성

```
tee /opt/elasticsearch/config/jvm.options<<EOF
-Xms1g
-Xmx1g

-XX:+UseG1GC

-Djava.io.tmpdir=${ES_TMPDIR}

20-:--add-modules=jdk.incubator.vector

23:-XX:CompileCommand=dontinline,java/lang/invoke/MethodHandle.setAsTypeCache
23:-XX:CompileCommand=dontinline,java/lang/invoke/MethodHandle.asTypeUncached

-Dorg.apache.lucene.store.defaultReadAdvice=normal
-Dorg.apache.lucene.store.MMapDirectory.sharedArenaMaxPermits=1

-XX:+HeapDumpOnOutOfMemoryError
-XX:+ExitOnOutOfMemoryError
-XX:HeapDumpPath=/data/elasticsearch/dump
-XX:ErrorFile=hs_err_pid%p.log
-Xlog:gc*,gc+age=trace,safepoint:file=gc.log:utctime,level,pid,tags:filecount=32,filesize=64m
EOF
```

### 3.2.3 elasticsearcy.yml 작성

- es01.wbl.com
```
tee /opt/elasticsearch/config/elasticsearch.yml<<EOF
cluster.name: wbl-cluster
discovery.seed_hosts: ["es01.wbl.com", "es02.wbl.com", "es03.wbl.com"]
cluster.initial_master_nodes: ["es01.wbl.com", "es02.wbl.com", "es03.wbl.com"]

node.name: es01.wbl.com
network.host: 192.168.56.121

path.data: /data/elasticsearch/data
path.logs: /data/elasticsearch/logs

path.repo:
  - /data/nfsshared/es_snapshots

xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.http.ssl.enabled: true

xpack.security.transport.ssl.keystore.path: /opt/elasticsearch/config/certs/es01-transport.p12
xpack.security.transport.ssl.certificate_authorities: ["/opt/elasticsearch/config/certs/elastic-ca.crt"]

xpack.security.http.ssl.keystore.path: /opt/elasticsearch/config/certs/http-wildcard.p12
EOF
```

- es02.wbl.com
```
tee /opt/elasticsearch/config/elasticsearch.yml<<EOF
cluster.name: wbl-cluster
discovery.seed_hosts: ["es01.wbl.com", "es02.wbl.com", "es03.wbl.com"]
cluster.initial_master_nodes: ["es01.wbl.com", "es02.wbl.com", "es03.wbl.com"]

node.name: es02.wbl.com
network.host: 192.168.56.122

path.data: /data/elasticsearch/data
path.logs: /data/elasticsearch/logs

path.repo:
  - /data/nfsshared/es_snapshots

# X-Pack Security (Basic)
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.http.ssl.enabled: true

# Transport TLS
xpack.security.transport.ssl.keystore.path: /opt/elasticsearch/config/certs/es01-transport.p12
#xpack.security.transport.ssl.keystore.password:
#xpack.security.transport.ssl.truststore.path: /opt/elasticsearch/config/certs/elastic-ca.p12
xpack.security.transport.ssl.certificate_authorities: ["/opt/elasticsearch/config/certs/elastic-ca.crt"]
#xpack.security.transport.ssl.truststore.password:

# HTTP TLS
xpack.security.http.ssl.keystore.path: /opt/elasticsearch/config/certs/http-wildcard.p12
#xpack.security.http.ssl.keystore.password:
#xpack.security.http.ssl.truststore.path: /opt/elasticsearch/config/certs/elastic-ca.p12
#xpack.security.http.ssl.truststore.password:
EOF
```

- es03.wbl.com
```
tee /opt/elasticsearch/config/elasticsearch.yml<<EOF
cluster.name: wbl-cluster
discovery.seed_hosts: ["es01.wbl.com", "es02.wbl.com", "es03.wbl.com"]
cluster.initial_master_nodes: ["es01.wbl.com", "es02.wbl.com", "es03.wbl.com"]

node.name: es03.wbl.com
network.host: 192.168.56.123

path.data: /data/elasticsearch/data
path.logs: /data/elasticsearch/logs

path.repo:
  - /data/nfsshared/es_snapshots

# X-Pack Security (Basic)
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.http.ssl.enabled: true

# Transport TLS
xpack.security.transport.ssl.keystore.path: /opt/elasticsearch/config/certs/es01-transport.p12
#xpack.security.transport.ssl.keystore.password:
#xpack.security.transport.ssl.truststore.path: /opt/elasticsearch/config/certs/elastic-ca.p12
xpack.security.transport.ssl.certificate_authorities: ["/opt/elasticsearch/config/certs/elastic-ca.crt"]
#xpack.security.transport.ssl.truststore.password:

# HTTP TLS
xpack.security.http.ssl.keystore.path: /opt/elasticsearch/config/certs/http-wildcard.p12
#xpack.security.http.ssl.keystore.password:
#xpack.security.http.ssl.truststore.path: /opt/elasticsearch/config/certs/elastic-ca.p12
#xpack.security.http.ssl.truststore.password:
EOF
```

### 3.2.4 ca 및 cert 생성

- es01.wbl.com
```
#cert 디렉토리 생성
mkdir -p /opt/elasticsearch/config/certs

#ca cert 생성(비밀번호없이)
```
bin/elasticsearch-certutil ca \
  --out config/certs/elastic-ca.p12 \
  --pass ""

# ca.crt 포맷 파일 생성
cd config/certs/
openssl pkcs12 -in elastic-ca.p12 -nokeys -out elastic-ca.crt -passin pass:

# HTTPS 와일드카드 인증서 생성
cd /opt/elasticsearch

bin/elasticsearch-certutil cert \
  --ca config/certs/elastic-ca.p12 \
  --ca-pass "" \
  --name http-wildcard \
  --dns *.wbl.com \
  --out config/certs/http-wildcard.p12 \
  --pass ""

# es01용 transport cert 파일 생성
bin/elasticsearch-certutil cert \
  --ca config/certs/elastic-ca.p12 \
  --ca-pass "" \
  --name es01.wbl.com \
  --ip 192.168.56.121 \
  --dns es01.wbl.com \
  --out config/certs/es01-transport.p12 \
  --pass ""

# es02.wbl.com 전송
scp config/certs es02.wbl.com:/opt/elasticsearch/config
```

- es02.wbl.com
```
cd /opt/elasticsearch

# es02용 transport cert 파일 생성
bin/elasticsearch-certutil cert \
  --ca config/certs/elastic-ca.p12 \
  --ca-pass "" \
  --name es02.wbl.com \
  --ip 192.168.56.122 \
  --dns es02.wbl.com \
  --out config/certs/es02-transport.p12 \
  --pass ""

# es03.wbl.com 전송
scp config/certs es03.wbl.com:/opt/elasticsearch/config
```

- es03.wbl.com
```
cd /opt/elasticsearch

# es03용 transport cert 파일 생성
bin/elasticsearch-certutil cert \
  --ca config/certs/elastic-ca.p12 \
  --ca-pass "" \
  --name es03.wbl.com \
  --ip 192.168.56.123 \
  --dns es03.wbl.com \
  --out config/certs/es03-transport.p12 \
  --pass ""

# es01.wbl.com & es02.wbl.com 전송
scp config/certs es01.wbl.com:/opt/elasticsearch/config
scp config/certs es02.wbl.com:/opt/elasticsearch/config
```

### 3.2.3 systemd 자동실행 등록

```
sudo tee /etc/systemd/system/elasticsearch.service<<EOF
[Unit]
Description=elasticsearch
After=network.target

[Service]
User=wbl
Type=simple
LimitNOFILE=65536
LimitNPROC=4096
LimitMEMLOCK=infinity
Environment="ES_JAVA_HOME=/opt/jdk"
ExecStart=/opt/elasticsearch/bin/elasticsearch
KillMode=process
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
```

### 3.2.4 커널 설정

```
sudo tee /etc/sysctl.d/99-elasticsearch.conf<<EOF
vm.max_map_count=262144
EOF

# 설정반영
sudo sysctl --system
```

### 3.2.5 실행

```
sudo systemctl daemon-reload
sudo systemctl enable elasticsearch
sudo systemctl start elasticsearch
```

#########################################################################

## 3.3 Kibana 설치/구성
- 대상: mon01.wbl.com

### 3.3.1 환경구성

```
# 패키지 다운로드
wget https://artifacts.elastic.co/downloads/kibana/kibana-9.2.1-linux-x86_64.tar.gz
tar zxf kibana-9.2.1-linux-x86_64.tar.gz
sudo mv kibana-9.2.1 /opt/
sudo ln -sf /opt/kibana-9.2.1 /opt/kibana

# node option
tee /opt/kibana/config/node.options<<EOF
--max-old-space-size=1024
--unhandled-rejections=warn
--dns-result-order=ipv4first
--no-experimental-require-module
EOF
```

### 3.3.2 kibana_system 및 elastic 계정 비밀번호 설정

- es01.wbl.com 접속
```
cd /opt/elasticsearch
# kibana_system
bin/elasticsearch-reset-password -u kibana_system

# elastic
bin/elasticsearch-reset-password -u elastic

# 인증서 kibana 전송
cd /opt/elasticsearch/config
scp -r certs mon01.wbl.com:/opt/kibana/config
```

### 3.3.3 kibana.yml 작성

```
tee /opt/kibana/config/kibana.yml<<EOF
server.port: 5601
server.host: "0.0.0.0"

server.name: "mon.wbl.com"

elasticsearch.hosts: ["https://es01.wbl.com:9200"]

elasticsearch.username: "kibana_system"
elasticsearch.password: "1q2w3e4r5t"

elasticsearch.ssl.certificateAuthorities: ["/opt/kibana/config/certs/elastic-ca.crt"]
elasticsearch.ssl.verificationMode: full
EOF
```

### 3.3.4 systemd 자동실행 등록

```
sudo systemctl daemon-reload
sudo systemctl enable kibana
sudo systemctl start kibana
```

#########################################################################

## 3.4 Logstash 설치
- 대상: mon01.wbl.com

### 3.4.1 패키지 다운로드

```
# 패키지다운로드
wget https://artifacts.elastic.co/downloads/logstash/logstash-9.2.1-linux-x86_64.tar.gz
tar zxf logstash-9.2.1-linux-x86_64.tar.gz
mv logstash-9.2.1 /opt/logstash-9.2.1
ln -sf /opt/logstash-9.2.1 /opt/logstash

### 3.4.2 logstash-syslog 구성

- 디렉토리 생성
```
mkdir -p /data/logstash-syslog/config
mkdir -p /data/logstash-syslog/data
mkdir -p /data/logstash-syslog/logs
```

- 스크립트 작성
```
tee /data/logstash-syslog/config/logstash-syslog.conf<<EOF
input {
  kafka {
    bootstrap_servers => "192.168.56.121:9092,192.168.56.122:9092,192.168.56.123:9092"
    topics            => ["syslog-messages"]

    # td-agent가 JSON으로 전송하므로 JSON codec 사용
    codec             => "json"

    # Kafka consumer 설정
    group_id          => "logstash-syslog-group"
    auto_offset_reset => "latest"
  }
}

filter {
  # grok/mutate/json/etc
}

output {
  elasticsearch {
    hosts    => ["https://es01.wbl.com:9200"]

    # 인증 정보
    user     => "elastic"
    password => "1q2w3e4r5t"

    # 인덱스 네이밍
    index => "system-messages-%{+YYYY.MM.dd}"

    # Elasticsearch 8.x TLS 검증 비활성화
#    ssl                                  => true
    ssl_enabled => true
#    ssl_certificate_verification         => false
    ssl_verification_mode => "none"
    # 필요하면 아래 사용:
#    cacert => "/opt/kibana/config/certs/elastic-ca.crt"
    ilm_enabled => false
  }

# Debuging
#  stdout {
#    codec => rubydebug
#  }
}
EOF
```

- 자동실행 등록
```
sudo tee /etc/systemd/system/logstash-syslog<<EOF
[Unit]
Description=logstash-syslog
After=network-online.target

[Service]
User=wbl
Group=wbl
Environment="LOGSTASH_HOME=/data/logstash-syslog"
#ExecStart=/opt/logstash/bin/logstash -r -f /data/logstash-syslog/config/logstash-syslog.conf
ExecStart=/opt/logstash/bin/logstash \
  --path.data /data/logstash-syslog/data \
  --path.logs /data/logstash-syslog/logs \
  -f /data/logstash-syslog/config/logstash-syslog.conf

KillMode=process
Restart=on-failure

[Install]
WantedBy=multi-user.target
Alias=logstash-syslog.service
EOF
```

- 실행
```
sudo systemctl daemon-reload
sudo systemctl enable logstash-syslog
sudo systemctl start logstash-syslog
```

### 3.4.3 logstash-nginx 구성

- 디렉토리 생성
```
mkdir -p /data/logstash-nginx/config
mkdir -p /data/logstash-nginx/patterns
mkdir -p /data/logstash-nginx/data
mkdir -p /data/logstash-nginx/logs
```

- patterns 등록
```
tee /data/logstash-nginx/patterns/nginx<<EOF
NGINX_ACCESS %{IPORHOST:clientip} (?:-|(%{WORD}.%{WORD})) %{USER:ident} \[%{HTTPDATE:timestamp}\] "(?:%{WORD:verb} %{NOTSPACE:request}(?: HTTP/%{NUMBER:httpversion})?|%{DATA:rawrequest})" %{NUMBER:response} (?:%{NUMBER:bytes}|-) "%{QS:referrer}" "%{QS:agent}" "%{IPORHOST:x-forwarded-for}"
NGINX_ACCESS_EX  %{IPORHOST:clientip} (?:-|(%{WORD}.%{WORD})) %{USER:ident} \[%{HTTPDATE:timestamp}\] "(?:%{WORD:verb} %{NOTSPACE:request}(?: HTTP/%{NUMBER:httpversion})?|%{DATA:rawrequest})" %{NUMBER:response} (?:%{NUMBER:bytes}|-) "%{QS:referrer}" "%{QS:agent}" "%{IPORHOST:x-forwarded-for}" %{NUMBER:request_time}
EOF
```

- 스크립트 작성
```
tee /data/logstash-nginx/config/logstash-nginx.conf<<EOF
input {
  kafka {
    bootstrap_servers => "192.168.56.121:9092,192.168.56.122:9092,192.168.56.123:9092"
    type => "nginx"
    group_id => "nginx.access"
    topics => ["nginx.access"]
#    cunsumer_threads => 1
    decorate_events => true
#    auto_offset_reset => "earliest"
  }
}

filter {
  if [type] == "nginx" {
    urldecode {
#      field => "request"
      field => "message"
    }
    grok {
        patterns_dir => "/data/logstash-nginx/patterns/nginx"
        match => { "message" => "%{NGINX_ACCESS}" }
        remove_tag => ["_grokparsefailure"]
        add_tag => ["nginx_access"]
    }
#    geoip {
#      source => "clientip"
#    }
#    mutate {
#      remove_field => ["message"]
#    }
  }
}
#output { stdout { codec => rubydebug } }
output {
  elasticsearch {
    hosts => ["https://192.168.56.121:9200"]
    user     => "elastic"
    password => "1q2w3e4r5t"

    index => "nginx-access-%{+YYYY.MM.dd}"
#    document_type => "%{type}"
    ssl_enabled => true
#    ssl_certificate_verification         => false
    ssl_verification_mode => "none"
#    cacert => "/opt/kibana/config/certs/elastic-ca.crt"
    ilm_enabled => false

  }
}
EOF
```

- 자동실행 등록
```
sudo tee /etc/systemd/system/logstash-nginx.service<<EOF
[Unit]
Description=logstash-nginx
After=network-online.target

[Service]
User=wbl
Group=wbl
Environment="LOGSTASH_HOME=/data/logstash-nginx"
#ExecStart=/opt/logstash/bin/logstash -r -f /data/logstash-nginx/config/logstash-nginx.conf
ExecStart=/opt/logstash/bin/logstash \
  --path.data /data/logstash-nginx/data \
  --path.logs /data/logstash-nginx/logs \
  -f /data/logstash-nginx/config/logstash-nginx.conf

KillMode=process
Restart=on-failure

[Install]
WantedBy=multi-user.target
Alias=logstash-nginx.service
EOF
```

- 실행
```
sudo systemctl daemon-reload
sudo systemctl enable logstash-nginx
sudo systemctl start logstash-nginx
```

#########################################################################

## 3.5 td-agent 설치

### 3.5.1 패키지 설치

```
# td-agent 설치
curl -L https://toolbelt.treasuredata.com/sh/install-redhat-td-agent4.sh | sudo sh

# plugin 설치
sudo td-agent-gem install fluent-plugin-elasticsearch
sudo td-agent-gem install fluent-plugin-kafka
sudo td-agent-gem install fluent-plugin-record-reformer
sudo td-agent-gem install fluent-plugin-heroku-syslog
```

### 3.5.2 스크립트 작성
- mon01.wbl.com
```
tee /etc/td-agent/td-agent.conf<<EOF
##===========================================
## INPUT: /var/log/messages 수집
##===========================================
<source>
  @type tail
  path /var/log/messages
  pos_file /var/log/td-agent/messages.pos
  tag syslog.messages
  format syslog          # td-agent 코어 내장 syslog 파서
</source>

## (옵션) hostname, 에이전트 정보 추가
<filter syslog.**>
  @type record_transformer
  <record>
    hostname "#{Socket.gethostname}"
    agent    "td-agent"
  </record>
</filter>

##===========================================
## OUTPUT: Kafka 로 전송
##===========================================
<match syslog.**>
  @type kafka2
  # Kafka 브로커 리스트 (실제 환경에 맞게 수정)
  brokers 192.168.56.121:9092,192.168.56.122:9092,192.168.56.123:9092

  # 보낼 topic 이름
  default_topic syslog-messages

  # 메시지 포맷: Logstash에서 json으로 읽을 수 있게 JSON 사용
  <format>
    @type json
  </format>

  # 에러/재시도 관련 옵션 (필요시 조정)
  max_send_retries 3
  required_acks    -1
  compression_codec gzip
</match>

<source>
  @type tail
  path /var/log/nginx/access*.log
  pos_file /var/log/td-agent/access.log.pos
  tag nginx.access
  format /(?<message>.*)/
</source>

<match nginx.access>
  @type kafka2
  brokers 192.168.56.121:9092,192.168.56.122:9092,192.168.56.123:9092
  zookeeper 192.168.56.121:2181,192.168.56.122:2181,192.168.56.123:2181
  default_topic nginx.access
  <format>
    @type single_value
  </format>
</match>
EOF
```

- es01.wbl.com
```
tee /etc/td-agent/td-agent.conf<<EOF
##===========================================
## INPUT: /var/log/messages 수집
##===========================================
<source>
  @type tail
  path /var/log/messages
  pos_file /var/log/td-agent/messages.pos
  tag syslog.messages
  format syslog          # td-agent 코어 내장 syslog 파서
</source>

## (옵션) hostname, 에이전트 정보 추가
<filter syslog.**>
  @type record_transformer
  <record>
    hostname "#{Socket.gethostname}"
    agent    "td-agent"
  </record>
</filter>

##===========================================
## OUTPUT: Kafka 로 전송
##===========================================
<match syslog.**>
  @type kafka2
  # Kafka 브로커 리스트 (실제 환경에 맞게 수정)
  brokers 192.168.56.121:9092,192.168.56.122:9092,192.168.56.123:9092

  # 보낼 topic 이름
  default_topic syslog-messages

  # 키/파티션 전략 (기본값으로도 충분하지만 참고용)
  # default_partition_key  hostname
  # partition_key         hostname

  # 메시지 포맷: Logstash에서 json으로 읽을 수 있게 JSON 사용
  <format>
    @type json
  </format>

  # 에러/재시도 관련 옵션 (필요시 조정)
  max_send_retries 3
  required_acks    -1
  compression_codec gzip
</match>
EOF
```

- es02.wbl.com
```
tee /etc/td-agent/td-agent.conf<<EOF
##===========================================
## INPUT: /var/log/messages 수집
##===========================================
<source>
  @type tail
  path /var/log/messages
  pos_file /var/log/td-agent/messages.pos
  tag syslog.messages
  format syslog          # td-agent 코어 내장 syslog 파서
</source>

## (옵션) hostname, 에이전트 정보 추가
<filter syslog.**>
  @type record_transformer
  <record>
    hostname "#{Socket.gethostname}"
    agent    "td-agent"
  </record>
</filter>

##===========================================
## OUTPUT: Kafka 로 전송
##===========================================
<match syslog.**>
  @type kafka2
  # Kafka 브로커 리스트 (실제 환경에 맞게 수정)
  brokers 192.168.56.121:9092,192.168.56.122:9092,192.168.56.123:9092

  # 보낼 topic 이름
  default_topic syslog-messages

  # 키/파티션 전략 (기본값으로도 충분하지만 참고용)
  # default_partition_key  hostname
  # partition_key         hostname

  # 메시지 포맷: Logstash에서 json으로 읽을 수 있게 JSON 사용
  <format>
    @type json
  </format>

  # 에러/재시도 관련 옵션 (필요시 조정)
  max_send_retries 3
  required_acks    -1
  compression_codec gzip
</match>
EOF
```

- es03.wbl.com
```
tee /etc/td-agent/td-agent.conf<<EOF
##===========================================
## INPUT: /var/log/messages 수집
##===========================================
<source>
  @type tail
  path /var/log/messages
  pos_file /var/log/td-agent/messages.pos
  tag syslog.messages
  format syslog          # td-agent 코어 내장 syslog 파서
</source>

## (옵션) hostname, 에이전트 정보 추가
<filter syslog.**>
  @type record_transformer
  <record>
    hostname "#{Socket.gethostname}"
    agent    "td-agent"
  </record>
</filter>

##===========================================
## OUTPUT: Kafka 로 전송
##===========================================
<match syslog.**>
  @type kafka2
  # Kafka 브로커 리스트 (실제 환경에 맞게 수정)
  brokers 192.168.56.121:9092,192.168.56.122:9092,192.168.56.123:9092

  # 보낼 topic 이름
  default_topic syslog-messages

  # 키/파티션 전략 (기본값으로도 충분하지만 참고용)
  # default_partition_key  hostname
  # partition_key         hostname

  # 메시지 포맷: Logstash에서 json으로 읽을 수 있게 JSON 사용
  <format>
    @type json
  </format>

  # 에러/재시도 관련 옵션 (필요시 조정)
  max_send_retries 3
  required_acks    -1
  compression_codec gzip
</match>
EOF
```

### 3.5.3 systemd 자동실행 등록
```
# td-agent.service 실행계정변경
# vi /usr/lib/systemd/system/td-agent.service
# [Service]
# User=td-agent => User=root
# Group=td-agent => Group=root

sudo systemctl daemon-reload
sudo systemctl enable td-agent
sudo systemctl start td-agent
```

## 3.6 nginx 설치 및 구성
- 대상: mon01.wbl.com
```
# nginx 설치
sudo dnf install nginx -y

# nginx.conf 수정 변경
sudo tee /etc/nginx/nginx.conf<<EOF
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

include /usr/share/nginx/modules/*.conf;

events {
    worker_connections 1024;
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for" $request_time';

    access_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 4096;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    include /etc/nginx/conf.d/*.conf;
}
EOF

# webhook 테스트
sudo tee /etc/nginx/conf.d/webhook.conf<<EOF
server {
    listen 8080;
    server_name _;

    access_log /var/log/nginx/access-webhook.log;

    location /event {
        # Webhook 수신 시 단순 응답
        return 200 "webhook received\n";
    }
}
EOF
sudo tee /etc/nginx/conf.d/monitoring.conf<EOF
server {
    server_name localhost;
    server_tokens off;
    listen 127.0.0.1:19090;
    location /metrics {
        if ($request_method !~ ^(GET|POST)$) {
            return 403;
        }
        stub_status on;
#        dev_methods off;
    }
}
EOF

# 실행
sudo systemctl enable nginx
sudo systemctl start nginx
```

#########################################################################

# 4. prometheus 모니터링 구성

## 4.1 mysqld 설치
- mon01.wbl.com

### 4.1.1 설치
```
sudo dnf install mysql-server -y
```

### 4.1.2 실행 및 root 계정변경
```
mysql --host=127.0.0.1 --port=3306 -u root
###########################
# root 비밀번호 변경
alter user root identified by '!q2w3e4r5T';

# grafana 계정 생성
create user grafana@'%' identified by '1q2w#e4r5T';

# grafana database 생성
create database grafana;

grant all privileges on grafana.* to grafana@'%' with grant option;
flush privileges;
```

### 4.1.3 mysqld_exporter 계정생성

```
mysql --host=127.0.0.1 --port=3306 -u root -p mysql

CREATE USER 'exporter'@'localhost' IDENTIFIED BY '1q2w#e4r5T' WITH MAX_USER_CONNECTIONS 3;
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'exporter'@'localhost';

flush privileges
```

#########################################################################

## 4.2 exporter 설치
- 대상: mon01.wbl.com

### 4.2.1 패키지 다운로드

```
mkdir ~/download
cd ~/download
wget https://github.com/prometheus/prometheus/releases/download/v3.8.0/prometheus-3.8.0.linux-amd64.tar.gz
wget https://github.com/prometheus/alertmanager/releases/download/v0.29.0/alertmanager-0.29.0.linux-amd64.tar.gz
wget https://github.com/prometheus/mysqld_exporter/releases/download/v0.18.0/mysqld_exporter-0.18.0.linux-amd64.tar.gz
wget https://github.com/prometheus/node_exporter/releases/download/v1.10.2/node_exporter-1.10.2.linux-amd64.tar.gz
wget https://github.com/ncabatoff/process-exporter/releases/download/v0.8.7/process-exporter-0.8.7.linux-amd64.tar.gz
wget https://github.com/nginx/nginx-prometheus-exporter/releases/download/v1.5.1/nginx-prometheus-exporter_1.5.1_linux_amd64.tar.gz
wget https://github.com/dabealu/zookeeper-exporter/releases/download/v0.1.13/zookeeper-exporter-v0.1.13-linux.tar.gz
wget https://github.com/danielqsj/kafka_exporter/releases/download/v1.9.0/kafka_exporter-1.9.0.linux-amd64.tar.gz
wget https://github.com/prometheus-community/elasticsearch_exporter/releases/download/v1.9.0/elasticsearch_exporter-1.9.0.linux-amd64.tar.gz

```

### 4.2.2 패키지 구성
```
sudo mkdir -p /opt/prometheus/bin
sudo mkdir -p /opt/prometheus/conf
sudo mkdir -p /opt/prometheus/data
sudo mkdir -p /opt/prometheus/consoles
sudo mkdir -p /opt/prometheus/console_libraries
sudo chown -R wbl:wbl /opt/prometheus

tar zxf prometheus-3.8.0.linux-amd64.tar.gz
mv prometheus-3.8.0.linux-amd64/prometheus /opt/prometheus/bin
mv prometheus-3.8.0.linux-amd64/promtool /opt/prometheus/bin
rm -rf prometheus-3.8.0.linux-amd64

tar zxf alertmanager-0.29.0.linux-amd64.tar.gz
mv alertmanager-0.29.0.linux-amd64/alertmanager /opt/prometheus/bin
mv alertmanager-0.29.0.linux-amd64/amtool /opt/prometheus/bin
rm -rf alertmanager-0.29.0.linux-amd64

tar zxf mysqld_exporter-0.18.0.linux-amd64.tar.gz
mv mysqld_exporter-0.18.0.linux-amd64/mysqld_exporter /opt/prometheus/bin
rm -rf mysqld_exporter-0.18.0.linux-amd64

tar zxf node_exporter-1.10.2.linux-amd64.tar.gz
mv node_exporter-1.10.2.linux-amd64/node_exporter /opt/prometheus/bin
rm -rf node_exporter-1.10.2.linux-amd64

tar zxf process-exporter-0.8.7.linux-amd64.tar.gz
mv process-exporter-0.8.7.linux-amd64/process-exporter /opt/prometheus/bin
rm -rf process-exporter-0.8.7.linux-amd64/process-exporter

tar zxf nginx-prometheus-exporter_1.5.1_linux_amd64.tar.gz
mv nginx-prometheus-exporter /opt/prometheus/bin

tar zxf zookeeper-exporter-v0.1.13-linux.tar.gz
mv zookeeper-exporter-v0.1.13-linux/zookeeper-exporter /opt/prometheus/bin

tar zxf kafka_exporter-1.9.0.linux-amd64.tar.gz
mv kafka_exporter-1.9.0.linux-amd64/kafka_exporter /opt/prometheus/bin

tar zxf elasticsearch_exporter-1.9.0.linux-amd64.tar.gz
mv elasticsearch_exporter-1.9.0.linux-amd64/elasticsearch_exporter /opt/prometheus/bin

scp -r /opt/prometheus/bin es01.wbl.com:/opt/prometheus
scp -r /opt/prometheus/bin es02.wbl.com:/opt/prometheus
scp -r /opt/prometheus/bin es03.wbl.com:/opt/prometheus
```

4.2.3 node_exporter
- 대상: mon01.wbl.com es01.wbl.com es02.wbl.com es03.wbl.com
- node_exporter.conf 작성
```
tee /opt/prometheus/conf/node_exporter.conf<<EOF
OPTIONS="--collector.textfile.directory /opt/prometheus/data"
EOF
```
- systemd 자동실행 등록
```
sudo tee /etc/systemd/system/node_exporter.service<<EOF
[Unit]
Description=node_exporter
After=network.target

[Service]
User=wbl
Restart=on-failure
Type=simple
EnvironmentFile=/opt/prometheus/conf/node_exporter.conf
ExecStart=/opt/prometheus/bin/node_exporter

[Install]
WantedBy=multi-user.target
EOF
```
- 실행
```
sudo systemctl daemon-reload
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
```

4.2.4 process-exporter
- 대상: mon01.wbl.com es01.wbl.com es02.wbl.com es03.wbl.com
- process-exporter.yaml 작성
```
tee /opt/prometheus/conf/process-exporter.yaml<<EOF
process_names:
  - comm:
    - chronyd
    - node_exporter
  - exe:
    - /opt/prometheus/bin/process-exporter
  - name: "elasticsearch"
    cmdline:
      - 'org.elasticsearch.bootstrap.Elasticsearch'
  - name: "kafka"
    cmdline:
    - '/opt/kafka/config/server.properties'
  - name: "zookeeper"
    cmdline:
    - '/opt/zookeeper/conf/zoo.cfg'
EOF
```
- systemd 자동실행 등록
```
sudo tee /etc/systemd/system/process-exporter.service<<EOF
[Unit]
Description=process-exporter
After=network.target

[Service]
User=wbl
Restart=on-failure
Type=simple
ExecStart=/opt/prometheus/bin/process-exporter -config.path /opt/prometheus/conf/process-exporter.yaml

[Install]
WantedBy=multi-user.target
EOF
```
- 실행
```
sudo systemctl daemon-reload
sudo systemctl enable process-exporter
sudo systemctl start process-exporter
```

4.2.5 mysqld_exporter
- 대상: mon01.wbl.com
- my.conf 작성
```
tee /opt/prometheus/conf/my.conf<<EOF
[client]
user = exporter
password = 1q2w#e4r5T
host = localhost
port = 3306

[client.servers]
user = exporter
password = 1q2w#e4r5T
host = localhost
port - 3306
EOF
```
- systemd 자동실행 등록
```
sudo tee /etc/systemd/system/mysqld_exporter.service<<EOF
[Unit]
Description=Prometheus MySQL Exporter
After=network.target

[Service]
User=wbl
Group=wbl

Type=simple
Environment='DATA_SOURCE_NAME=exporter:password@tcp(192.168.56.120:3306)/'
ExecStart=/opt/prometheus/bin/mysqld_exporter --config.my-cnf /opt/prometheus/conf/my.conf   --collect.global_status   --collect.info_schema.innodb_metrics   --collect.auto_increment.columns   --collect.binlog_size   --collect.global_variables   --collect.info_schema.tablestats   --collect.info_schema.query_response_time   --collect.info_schema.userstats   --collect.info_schema.tables   --collect.perf_schema.tablelocks   --collect.perf_schema.file_events   --collect.perf_schema.eventswaits   --collect.perf_schema.indexiowaits   --collect.perf_schema.tableiowaits   --collect.slave_status
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
```
- 실행
```
sudo systemctl daemon-reload
sudo systemctl enable mysqld_exporter
sudo systemctl start mysqld_exporter
```

4.2.7 zookeeper-exporter
- 대상: mon01.wbl.com
- systemd 자동실행 등록
```
sudo tee /etc/systemd/system/zookeeper-exporter.service<<EOF
[Unit]
Description=zookeeper-exporter
After=network.target

[Service]
User=wbl
Restart=on-failure
Type=simple
ExecStart=/opt/prometheus/bin/zookeeper-exporter -listen=192.168.56.120:9141 -zk-hosts='192.168.56.121:2181,192.168.56.122:2181,192.168.56.123:2181'

[Install]
WantedBy=multi-user.target
EOF
```
- 실행
```
sudo systemctl daemon-reload
sudo systemctl enable zookeeper-exporter
sudo systemctl start zookeeper-exporter
```

4.2.8 kafka_exporter
- 대상: es01.wbl.com
- systemd 자동실행 등록
```
sudo tee /etc/systemd/system/kafka_exporter.service<<EOF
[Unit]
Description=kafka_exporter
After=network.target

[Service]
User=wbl
Restart=on-failure
Type=simple
ExecStart=/opt/prometheus/bin/kafka_exporter --kafka.server=192.168.56.121:9092

[Install]
WantedBy=multi-user.target
EOF
```

- 대상: es02.wbl.com
- systemd 자동실행 등록
```
sudo tee /etc/systemd/system/kafka_exporter.service<<EOF
[Unit]
Description=kafka_exporter
After=network.target

[Service]
User=wbl
Restart=on-failure
Type=simple
ExecStart=/opt/prometheus/bin/kafka_exporter --kafka.server=192.168.56.122:9092

[Install]
WantedBy=multi-user.target
EOF
```

- 대상: es03.wbl.com
- systemd 자동실행 등록
```
sudo tee /etc/systemd/system/kafka_exporter.service<<EOF
[Unit]
Description=kafka_exporter
After=network.target

[Service]
User=wbl
Restart=on-failure
Type=simple
ExecStart=/opt/prometheus/bin/kafka_exporter --kafka.server=192.168.56.123:9092

[Install]
WantedBy=multi-user.target
EOF
```

- 실행
```
sudo systemctl daemon-reload
sudo systemctl enable kafka_exporter
sudo systemctl start kafka_exporter
```


4.2.9 elasticsearch_exporter
- 대상: es01.wbl.com
- systemd 자동실행 등록
```
sudo tee /etc/systemd/system/elasticsearch_exporter.service<<EOF
[Unit]
Description=elasticsearch_exporter
After=network.target

[Service]
User=wbl
Restart=on-failure
Type=simple
ExecStart=/opt/prometheus/bin/elasticsearch_exporter \
       --web.listen-address=192.168.56.121:9114 \
       --es.uri="https://elastic:1q2w3e4r5t@es01.wbl.com:9200" \
       --es.ca=/opt/elasticsearch/config/certs/elastic-ca.crt

[Install]
WantedBy=multi-user.target
EOF
```

- 대상: es02.wbl.com
- systemd 자동실행 등록
```
sudo tee /etc/systemd/system/elasticsearch_exporter.service<<EOF
[Unit]
Description=elasticsearch_exporter
After=network.target

[Service]
User=wbl
Restart=on-failure
Type=simple
ExecStart=/opt/prometheus/bin/elasticsearch_exporter \
       --web.listen-address=192.168.56.122:9114 \
       --es.uri="https://elastic:1q2w3e4r5t@es02.wbl.com:9200" \
       --es.ca=/opt/elasticsearch/config/certs/elastic-ca.crt

[Install]
WantedBy=multi-user.target

EOF
```

- 대상: es03.wbl.com
- systemd 자동실행 등록
```
sudo tee /etc/systemd/system/elasticsearch_exporter.service<<EOF
[Unit]
Description=elasticsearch_exporter
After=network.target

[Service]
User=wbl
Restart=on-failure
Type=simple
ExecStart=/opt/prometheus/bin/elasticsearch_exporter \
       --web.listen-address=192.168.56.123:9114 \
       --es.uri="https://elastic:1q2w3e4r5t@es03.wbl.com:9200" \
       --es.ca=/opt/elasticsearch/config/certs/elastic-ca.crt

[Install]
WantedBy=multi-user.target

EOF
```

- 실행
```
sudo systemctl daemon-reload
sudo systemctl enable elasticsearch_exporter
sudo systemctl start elasticsearch_exporter
```

## 4.3 prometheus 서버 구성
- 대상: mon01.wbl.com

### 4.3.1 prometheus.conf 작성

```
tee /opt/prometheus/conf/prometheus.yml<<EOF
# my global config
global:
  scrape_interval:     15s
  evaluation_interval: 15s

# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      - 192.168.56.120:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  - "/opt/prometheus/conf/rules.yml"
  # - "second_rules.yml"

scrape_configs:
  # The job name is added as a label  to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
    - targets: ['localhost:9090']
#  - job_name: 'aucturator'
#    metrics_path: '/actuator/prometheus'
#   static_configs:
#   - targets: ['192.168.56.121:8082','192.168.56.122:8083','192.168.56.123:8084']
  - job_name: 'my_monitoring'
    scrape_interval: '15s'
    file_sd_configs:
    - files:
      - 'conf.d/*.yml'
EOF
```

### 4.3.2 scrapping 추가
- node_exporter 
```
tee /opt/prometheus/conf/conf.d/nodes.yml<<EOF
- targets: ['192.168.56.120:9100','192.168.56.121:9100','192.168.56.122:9100','192.168.56.123:9100']
  labels:
    job: 'nodes'
EOF
```

- process-exporter 
```
tee /opt/prometheus/conf/conf.d/process.yml<<EOF
- targets: ['192.168.56.120:9256','192.168.56.121:9256','192.168.56.122:9256','192.168.56.123:9256']
  labels:
    job: 'processes'
EOF
```

- zookeeper-exporter 
```
tee /opt/prometheus/conf/conf.d/zookeeper.yml<<EOF
- targets: ['192.168.56.121:9141','192.168.56.122:9141','192.168.56.123:9141']
  labels:
    job: 'zookeeper'
EOF
```

- mysqld-exporter 
```
tee /opt/prometheus/conf/conf.d/mysqld.yml<<EOF
- targets: ['127.0.0.1:9104']
  labels:
    job: 'mysql'
EOF
```

- kafka-exporter 
```
tee /opt/prometheus/conf/conf.d/kafka.yml<<EOF
- targets: ['192.168.56.121:9308','192.168.56.122:9308','192.168.56.123:9308']
  labels:
    job: 'kafka'
EOF
```

- elasticsearch-exporter 
```
tee /opt/prometheus/conf/conf.d/elasticsearch.yml<<EOF
- targets: ['192.168.56.121:9114','192.168.56.122:9114','192.168.56.123:9114']
  labels:
    job: 'elasticsearch'
EOF
```

### 4.3.3 prometheus 자동실행 등록
- 대상: mon01.wbl.com

```
sudo tee /etc/systemd/system/prometheus.service<<EOF
[Unit]
Description=Prometheus
Documentation=https://prometheus.io/docs/introduction/overview/
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
Environment="GOMAXPROCS=1"
User=wbl
Group=wbl
ExecReload=/bin/kill -HUP
ExecStart=/opt/prometheus/bin/prometheus   --config.file=/opt/prometheus/conf/prometheus.yml   --storage.tsdb.path=/opt/prometheus/data   --web.console.templates=/opt/prometheus/consoles   --web.console.libraries=/opt/rometheus/console_libraries   --web.listen-address=0.0.0.0:9090   --web.enable-lifecycle   --web.external-url=

SyslogIdentifier=prometheus
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
```

- 실행
```
sudo systemctl daemon-reload
sudo systemctl enable prometheus
sudo systemctl start prometheus
```

## 4.4 alertmanager 구성

### 4.4.1 alertmanager.yml 작성

```
tee /opt/prometheus/conf/alertmanager.yml<<EOF
route:
  group_by: ['alertname']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 1h
  receiver: 'web.hook'
receivers:
  - name: 'web.hook'
    webhook_configs:
      - url: 'http://127.0.0.1:5001/'
inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'dev', 'instance']
EOF
```

### 4.4.2 자동실행 등록

```
sudo tee /etc/systemd/system/alertmanager.service<<EOF
[Unit]
Description=Prometheus Alertmanager.
Documentation=https://github.com/prometheus/alertmanager
After=network.target

[Service]
User=wbl
Group=wbl
ExecStart=/opt/prometheus/bin/alertmanager           --config.file=/opt/prometheus/conf/alertmanager.yml           --storage.path=/opt/prometheus/data
ExecReload=/bin/kill -HUP
Restart=on-failure
RestartSec=5s
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```
실행
```
sudo systemctl daemon-reload
sudo systemctl enable alertmanager
sudo systemctl start alertmanager
```

#########################################################################

# 5. Jaeger /w OTEL on Elasticsearch

## 5.1 OTEL Collector 구성

### 5.1.1 환경구성

```
curl --proto '=https' --tlsv1.2 -fOL https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/v0.141.0/otelcol_0.141.0_linux_amd64.tar.gz
tar -xvf otelcol_0.141.0_linux_amd64.tar.gz

sudo mkdir -p /opt/otel/bin
sudo mkdir -p /opt/otel/conf
sudo chown -R wbl:wbl /opt/otel

mv otelcol /opt/otel/bin
```

### 5.1.2 구성파일 작성

```
tee /opt/otel/conf/config.yml<<EOF
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317  # openapi/javaagent가 보내는 포트

processors:
  batch: {}

exporters:
  otlphttp:
    endpoint: "http://127.0.0.1:4318"
    tls:
      insecure: true

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlphttp]
EOF

```

### 5.1.3 자동실행 등록

```
sudo tee /etc/systemd/system/otelcol.service<<EOF
[Unit]
Description=otel-collector
After=network.target

[Service]
User=wbl
ExecStart=/opt/otel/bin/otelcol --config=/opt/otel/conf/config.yml
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
```
실행
```
sudo systemctl daemon-reload
sudo systemctl enable otelcol
sudo systemctl start otelcol
```

## 5.2 jaeger 구성

- 대상: mon01.wbl.com

### 5.2.1 경로 구성 및 다운로드

```
wget https://download.jaegertracing.io/v1.76.0/jaeger-1.76.0-linux-amd64.tar.gz

sudo mkdir -p /opt/jaeger/bin
sudo mkdir -p /opt/jaeger/conf
sudo chown -R wbl:wbl /opt/jaeger

tar zxf jaeger-1.76.0-linux-amd64.tar.gz
mv jaeger-1.76.0-linux-amd64/jaeger-all-in-one /opt/jaeger/bin
```

### 5.2.2 자동실행 등록

```
sudo tee /etc/systemd/system/jaeger.service<<EOF
[Unit]
Description=Jaeger all-in-one (collector + query + UI)
After=network.target

[Service]
User=wbl
Group=wbl
Restart=on-failure

# --- Elasticsearch 저장소 설정 ---
Environment=SPAN_STORAGE_TYPE=elasticsearch
Environment=ES_SERVER_URLS=https://es01.wbl.com:9200,https://es02.wbl.com:9200,https://es03.wbl.com:9200
Environment=ES_USERNAME=elastic
Environment=ES_PASSWORD=1q2w3e4r5t

# TLS 자체서명 인증서 환경 (테스트용: 검증 생략)
Environment=ES_TLS_ENABLED=true
Environment=ES_TLS_SKIP_HOST_VERIFY=true

# ES 9.x는 공식 지원 목록에 아직 없어서 8로 강제 지정 (API 호환 전제, 약간 실험적)
Environment=ES_VERSION=8

# --- Jaeger 실행 ---
ExecStart=/data/jaeger/bin/jaeger-all-in-one \
  --collector.otlp.enabled=true \
  --collector.otlp.grpc.host-port=0.0.0.0:4319 \
  --collector.otlp.http.host-port=0.0.0.0:4318 \
  --query.http-server.host-port=0.0.0.0:16686

[Install]
WantedBy=multi-user.target
EOF
```

## 5.3 javaagent 구성

- VM Host

### 5.3.1 환경구성

```
mkdir -p ~/openapi/otel
cd ~/openapi/otel
wget https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/latest/download/opentelemetry-javaagent.jar
```

### 5.3.2 agent 설정파일 작성
```
teel ~/openapi/otel/otel-config.properties<<EOF
otel.service.name=pub
otel.exporter.otlp.endpoint=http://192.168.56.120:4317
otel.exporter.otlp.protocol=grpc
otel.resource.attributes=service.name=openapi-pub
EOF

## 5.4 Test 프로그램 설정

```
mkdir ~/openapi/logs

# application.yml 설정
tee ~/openapi/application.yml<<EOF
info:
  app:
    name: openapi-pub
    version: 1.0.0
    description:  openapi-pub

endpoints:
  shutdown:
    enabled: true
    sensitive: false

management:
  security:
    enabled: false
  endpoints:
    web:
      exposure:
        include: metrics,health,prometheus
    health:
      show-details: always

spring:
  profiles:
    active: prod

---
spring:
  config:
    activate:
      on-profile: prod
  application:
    name: openapi-pub
logging:
  config: file:/home/wbl/openapi/logback-prod.xml

log:
  file:
    path: "/home/wbl/openapi"

auth:
  appVersion: 1.0.0
rest:
  connectionTimeout: 10000
  readTimeout: 10000

server:
  port: 8080
  tomcat:
    accesslog:
      enabled: true
      file-name: "access.log"
      file-date-format: ""
      roll-date: false
      tune-request-log-format: true
      pattern: "%h %l %u %{yyyy-MM-dd'Z'HH:mm:ss.SSS}t \"%m %U %H\" %s %b %D"
    basedir: "/home/wbl/openapi"

  servlet:
    context-path: /
    encoding:
      charset: UTF-8
      enabled: true
      force: true
---
EOF

# logback-prod.xml 작성
tee ~/openapi/logback-prod.xml<<EOF
<?xml version="1.0" encoding="UTF-8"?>

<configuration debug="false">
    <!-- VM argments에 home.path 추가하여 사용 -->
    <appender name="STDERR" class="ch.qos.logback.core.ConsoleAppender">
        <target>System.err</target>
        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
                <level>WARN</level>
        </filter>
                <encoder>
                <pattern>%-5level | %d{yyyy-MM-dd HH:mm.ss.SSS} | %logger{35}[%method:%line][%thread] - %msg%n</pattern>
        </encoder>
        </appender>
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <target>System.out</target>
        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
                <level>INFO</level>
        </filter>
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
                <level>WARN</level>
                <onMatch>DENY</onMatch>
        </filter>
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
                <level>ERROR</level>
                <onMatch>DENY</onMatch>
        </filter>
        <encoder>
            <pattern>%-5level | %d{yyyy-MM-dd HH:mm.ss.SSS} | %logger{35}[%method:%line][%thread] - %msg%n</pattern>
        </encoder>
    </appender>

    <appender name="appLogAppender" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${log.file.path:-/home/wbl/openapi}/logs/openapi-pub.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>${log.file.path:-/home/wbl/openapi}/logs/openapi-pub-%d{yyyy-MM-dd}.%i.log</fileNamePattern>
            <maxFileSize>${log.max.file.size:-64}MB</maxFileSize>
            <maxHistory>${log.max.history:-30}</maxHistory>
        </rollingPolicy>
        <encoder>
            <pattern>%-5level | %d{yyyy-MM-dd HH:mm.ss.SSS} | %logger{35}[%method:%line][%thread] - %msg%n</pattern>
        </encoder>
    </appender>
    <logger name="java.sql" additivity="false">
        <appender-ref ref="STDOUT" />
        <appender-ref ref="appLogAppender" />
        <appender-ref ref="STDERR" />
    </logger>
    <logger name="org.apache.http" additivity="false">
        <level value="INFO" />
        <appender-ref ref="STDOUT" />
        <appender-ref ref="STDERR" />
        <appender-ref ref="appLogAppender" />
    </logger>
    <!-- Spring 관련 로그 -->
    <logger name="org.springframework.boot" additivity="false">
        <level value="INFO" />
        <appender-ref ref="STDOUT" />
        <appender-ref ref="STDERR" />
        <appender-ref ref="appLogAppender" />
    </logger>
    <logger name="org.springframework.boot.web" additivity="false">
        <level value="INFO" />
        <appender-ref ref="STDOUT" />
        <appender-ref ref="STDERR" />
        <appender-ref ref="appLogAppender" />
    </logger>
    <logger name="org.springframework.boot.actuate" additivity="false">
        <level value="ERROR" />
        <appender-ref ref="STDOUT" />
        <appender-ref ref="STDERR" />
        <appender-ref ref="appLogAppender" />
    </logger>
    <logger name="com.zaxxer.hikari.pool" additivity="false">
        <level value="INFO" />
        <appender-ref ref="STDOUT" />
         <appender-ref ref="STDERR" />
        <appender-ref ref="appLogAppender" />
    </logger>
    <root level="${LOG_LEVEL:-INFO}">
        <appender-ref ref="STDOUT" />
        <appender-ref ref="STDERR" />
        <appender-ref ref="appLogAppender" />
    </root>
</configuration>
EOF

# 실행스크립트 
tee ~/openapi/start.sh<<EOF
#!/bin/bash

export JAVA_HOME="/opt/openjdk-17"
JARFILE="/home/wbl/openapi/openapi-pub-0.0.1.jar"

# OpenTelemetry 환경 변수
export OTEL_TRACES_EXPORTER=otlp
export OTEL_METRICS_EXPORTER=none
export OTEL_LOGS_EXPORTER=none
export OTEL_EXPORTER_OTLP_ENDPOINT=http://192.168.56.120:4317

/opt/openjdk-17/bin/java \
  -javaagent:/home/wbl/openapi/otel/opentelemetry-javaagent.jar \
  -Dotel.javaagent.configuration-file=/home/wbl/openapi/otel/otel-config.properties \
  -Djava.security.egd=file:/dev/./urandom \
  --add-opens java.base/java.lang=ALL-UNNAMED \
  -Dspring.config.location=/home/wbl/openapi/application.yml \
  -Duser.timezone=Asia/Seoul \
  -jar "$JARFILE"
EOF

# 실행권한
cd ~/openapi
chmod +x start.sh

# 실행
./start.sh

```
