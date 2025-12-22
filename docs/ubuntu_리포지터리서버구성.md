# td-agent repo setting
curl -o td-agent-apt-source.deb https://packages.treasuredata.com/4/ubuntu/jammy/pool/contrib/f/fluentd-apt-source/fluentd-apt-source_2020.8.25-1_all.deb
apt install -y ./td-agent-apt-source.deb
apt update

# grafana repo setting
sudo mkdir -p /etc/apt/keyrings/  
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null

echo  "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list

k8s_version=v1.34
curl -fsSL https://pkgs.k8s.io/core:/stable:/${k8s_version}/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/${k8s_version}/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list


apt update

# mk directories and give permission for deb packages
sudo apt update
sudo apt install -y acl

mkdir -p /data/offline-repo/debs
sudo setfacl -m u:_apt:rwx /data/offline-repo/debs
sudo setfacl -d -m u:_apt:rwx /data/offline-repo/debs

cd /data/offline-repo/debs

#download deb package 
apt download net-tools nfs-common nfs-kernel-server wget zip unzip tar chrony iputils-ping nginx mysql-server python3-pip ansible mysql-server td-agent grafana dpkg-dev file keepalived nmap tcpdump telnet traceroute git docker.io docker-compose apt-transport-https ca-certificates curl gpg  kubelet kubeadm kubectl acl vim
# deb dependencies

cd /data/offline-repo/
sudo apt install --download-only \
  --reinstall \
  --allow-downgrades \
  --allow-change-held-packages \
  -o=Dir::Cache::archives=./ \
net-tools nfs-common nfs-kernel-server wget zip unzip tar chrony iputils-ping nginx mysql-server python3-pip ansible mysql-server td-agent grafana dpkg-dev file keepalived nmap tcpdump telnet traceroute git docker.io docker-compose apt-transport-https ca-certificates curl gpg  kubelet kubeadm kubectl acl vim

#install dpkg-dev file
apt install dpkg-dev file nginx -y


sudo mkdir -p /repo/ubuntu/{dists/noble/main/binary-amd64,pool/main}
sudo mkdir -p /repo/files

sudo chown -R www-data:www-data /repo
sudo chmod -R 755 /repo


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
https://get.helm.sh/helm-v3.19.4-linux-amd64.tar.gz
https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
EOF


cd /repo/files
for f in $(cat /tmp/LIST);do wget $f;done


cp /data/offline-repo/debs/*.deb /repo/ubuntu/pool/main

cd /repo/ubuntu

dpkg-scanpackages pool /dev/null > dists/noble/main/binary-amd64/Packages
gzip -9fk dists/noble/main/binary-amd64/Packages

chown -R www-data:www-data /repo

rm -f /etc/nginx/sites-available/default
rm -f /etc/nginx/modules-enabled/default

sudo tee /etc/nginx/nginx.conf<<EOF
user www-data;
worker_processes auto;
pid /run/nginx.pid;
error_log /var/log/nginx/error.log;

events {
        worker_connections 768;
}

http {
        sendfile on;
        tcp_nopush on;
        types_hash_max_size 2048;

        include /etc/nginx/mime.types;
        default_type application/octet-stream;

        ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
        ssl_prefer_server_ciphers on;

        access_log /var/log/nginx/access.log;

        gzip on;

        include /etc/nginx/conf.d/*.conf;
}
EOF

tee /etc/nginx/conf.d/server.conf<<EOF
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name repo.local localhost _;

    root /repo;
    autoindex on;

    location / {
        try_files $uri $uri/ =404;
    }

    location /files/ {
        alias /repo/files/;
        autoindex on;
        autoindex_exact_size off;
        autoindex_localtime on;
        disable_symlinks if_not_owner from=/repo/files/;
        sendfile on;
        tcp_nopush on;
    }
}
EOF

systemctl restart nginx

##################################
## test
sudo rm -rf /etc/nginx/sites-enabled/*
sudo rm -rf /etc/nginx/sites-available/*

sudo tee /etc/apt/sources.list.d/wbl-monitor.list<<EOF
deb [trusted=yes] http://10.10.50.2/ubuntu noble main
EOF

apt clean
apt update
