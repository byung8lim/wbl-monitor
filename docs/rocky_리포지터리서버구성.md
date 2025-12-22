# td-agent repo setting
rpm --import https://packages.treasuredata.com/GPG-KEY-td-agent

sudo tee /etc/yum.repos.d/td.repo <<EOF;
[treasuredata]
name=TreasureData
baseurl=http://packages.treasuredata.com/4/redhat/\$releasever/\$basearch
gpgcheck=1
gpgkey=https://packages.treasuredata.com/GPG-KEY-td-agent
EOF



cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.34/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.34/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF



yum check-update

mkdir -p /data/download
cd /data/download
yum --downloadonly --downloaddir=. install grafana mysql-server td-agent python3-pip nginx net-tools nfs-utils wget zip unzip tar chrony createrepo haproxy keepalived docker -y


yum install nginx createrepo nginx

mkdir -p /var/www/html/rpmrepo/rocky/9/wbl-monitor/x86_64
mkdir /var/www/html/files

tee /tmp/LIST<<EOF
https://dlcdn.apache.org/zookeeper/zookeeper-3.8.5/apache-zookeeper-3.8.5-bin.tar.gz
https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-9.2.1-linux-x86_64.tar.gz
https://archive.apache.org/dist/kafka/3.8.1/kafka_2.13-3.8.1.tgz
https://artifacts.elastic.co/downloads/kibana/kibana-9.2.1-linux-x86_64.tar.gz
https://artifacts.elastic.co/downloads/logstash/logstash-9.2.1-linux-x86_64.tar.gz
https://github.com/prometheus/prometheus/releases/download/v3.8.0/prometheus-3.8.0.linux-amd64.tar.gz
https://github.com/prometheus/alertmanager/releases/download/v0.29.0/alertmanager-0.29.0.linux-amd64.tar.gz
https://github.com/prometheus/mysqld_exporter/releases/download/v0.18.0/mysqld_exporter-0.18.0.linux-amd64.tar.gz
https://github.com/prometheus/node_exporter/releases/download/v1.10.2/node_exporter-1.10.2.linux-amd64.tar.gz
https://github.com/ncabatoff/process-exporter/releases/download/v0.8.7/process-exporter-0.8.7.linux-amd64.tar.gz
https://github.com/nginx/nginx-prometheus-exporter/releases/download/v1.5.1/nginx-prometheus-exporter_1.5.1_linux_amd64.tar.gz
https://github.com/dabealu/zookeeper-exporter/releases/download/v0.1.13/zookeeper-exporter-v0.1.13-linux.tar.gz
https://github.com/danielqsj/kafka_exporter/releases/download/v1.9.0/kafka_exporter-1.9.0.linux-amd64.tar.gz
https://github.com/prometheus-community/elasticsearch_exporter/releases/download/v1.9.0/elasticsearch_exporter-1.9.0.linux-amd64.tar.gz
https://download.jaegertracing.io/v1.76.0/jaeger-1.76.0-linux-amd64.tar.gz
https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/latest/download/opentelemetry-javaagent.jar
https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/v0.141.0/otelcol_0.141.0_linux_amd64.tar.gz
https://builds.openlogic.com/downloadJDK/openlogic-openjdk/17.0.17+10/openlogic-openjdk-17.0.17+10-linux-x64.tar.gz
https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
EOF

cd /var/www/html/files
for f in $(cat /tmp/LIST);do wget $f;done


mv /data/download/*.rpm /var/www/html/rpmrepo/rocky/9/wbl-monitor/x86_64
createrepo /var/www/html/rpmrepo/rocky/9/wbl-monitor/x86_64

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

sudo tee /etc/nginx/conf.d/server.conf<<EOF
server {
    listen 80;
    server_name _;

    location /rpmrepo/ {
        alias /var/www/html/rpmrepo/;   # 실제 경로
        autoindex on;
        autoindex_exact_size off;
        autoindex_localtime on;
        try_files $uri $uri/ =404;
    }

    location /files/ {
        alias /var/www/html/files/;
        autoindex on;
        autoindex_exact_size off;
        autoindex_localtime on;
        try_files $uri $uri/ =404;
    }
}
EOF

sudo chown -R nginx /var/www

sudo systemctl restart nginx


######################################
# local host test

sudo rm -rf /etc/yum.repo.d/*

sudo tee /etc/yum.repo.d/local.repo<<EOF
[local-baseos]
name=Local wbl-monitor
baseurl=http://10.10.50.2/rpmrepo/rocky/9/wbl-monitor/x86_64/
enabled=1
gpgcheck=0
metadata_expire=1h
EOF

yum clean all
yum repolist

yum install td-agent 

