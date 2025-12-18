# 1. Control Host 초기화
```
mkdir -p ~/workspace
cd ~/workspace
```

# 2. Control Host용 필수 패키지 설치
```
sudo yum install python3-pip sshpass
sudo pip install ansible
```

# 3. wbl-monitor 다운로드 및 vars.yml 편집
 
# 4. 공통설치/elk 설치 전반부(playbook_part_one.yml)

## 4.1 주요 패키지 다운로드
```
ansible-playbook -i inventory.ini 00.preparing.yml
```

## 4.2 공통설치/elk 설치 전반부 설치
```
ansible-playbook -i inventory.ini playbook_part_one.yml
```

## 4.2. kibana_system password 변경
```
# es01.wbl.com 접속
# vars.yml의 kibana_elastic_password와 동일하게 설정해야함
cd /opt/elasticsearch
bin/elasticsearch-reset-password -u kibana_system --interactive --url https://es01.wbl.com:9200
```

## 4.2. elastic password 변경
# es01.wbl.com 접속
```
cd /opt/elasticsearch
# vars.yml의 elasticsearch_password 동일하게 설정해야함
bin/elasticsearch-reset-password -u elastic --interactive --url https://es01.wbl.com:9200
```

# 5. 후반부 설치
```
ansible-playbook -i inventory.ini playbook_part_two.yml
```
