### 1. Стек ELK

#### 1.1. Установка Elasticsearch

Установка Java
```
[root@elk ~]# yum install -y java-1.8.0-openjdk
```
Добавим репозиторий
```
[root@elk ~]# vi /etc/yum.repos.d/CentOS-ELK.repo

[elasticsearch-7.x]
name=ELK repository for 7.x packages
baseurl=https://artifacts.elastic.co/packages/7.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
```
Установка базы Elasticsearch

```
[root@elk ~]# yum -y install -y elasticsearch
[root@elk ~]# systemctl enable --now elasticsearch
```
Конфигурация Elasticsearch
```
[root@elk ~]# vi /etc/elasticsearch/elasticsearch.yml
network.host: 127.0.0.1
```
Настройка выделения памяти для работы базы
```
[root@elk ~]# vi /etc/elasticsearch/jvm.options
-Xms1g
-Xmx1g
```
```
[root@elk ~]# systemctl restart elasticsearch
```

#### 1.2. Установка Kibana
```
[root@elk ~]# yum install -y kibana
[root@elk ~]# systemctl enable --now kibana
```
Конфигурация Kibana
```
[root@elk ~]# vi /etc/kibana/kibana.yml
server.host: "192.168.11.103"
```
```
[root@elk ~]# systemctl restart kibana
```

#### 1.3. Установка Logstash
```
[root@elk ~]# yum install -y logstash
[root@elk ~]# systemctl enable --now logstash
```

Конфигурация данных logstash для лог nginx.
```
[root@elk ~]# vi /etc/logstash/conf.d/nginx_logs.conf
input {
  beats {
    port => 5044
  }
}

filter {
  if [service] == "access" {
    grok {
      match => { "message" => ["%{IPORHOST:[access][remote_ip]} - %{DATA:[access][user_name]} \[%{HTTPDATE:[access][time]}\] \"%{WORD:[access][method]} %{DATA:[access][url]} HTTP/%{NUMBER:[access][http_version]}\" %{NUMBER:[access][response_code]} %{NUMBER:[access][body_sent][bytes]} \"%{DATA:[access][referrer]}\" \"%{DATA:[access][agent]}\""] }
      remove_field => "message"
    }
    mutate {
      add_field => { "read_timestamp" => "%{@timestamp}" }
    }
    date {
      match => [ "[access][time]", "dd/MMM/YYYY:H:m:s Z" ]
      remove_field => "[access][time]"
    }
    useragent {
      source => "[access][agent]"
      target => "[access][user_agent]"
      remove_field => "[access][agent]"
    }
    geoip {
      source => "[access][remote_ip]"
      target => "[access][geoip]"
    }
  }
  else if [service] == "error" {
    grok {
      match => { "message" => ["%{DATA:[error][time]} \[%{DATA:[error][level]}\] %{NUMBER:[error][pid]}#%{NUMBER:[error][tid]}: (\*%{NUMBER:[error][connection_id]} )?%{GREEDYDATA:[error][message]}"] }
      remove_field => "message"
    }
    mutate {
      rename => { "@timestamp" => "read_timestamp" }
    }
    date {
      match => [ "[error][time]", "YYYY/MM/dd H:m:s" ]
      remove_field => "[error][time]"
    }
  }
}

output {
  elasticsearch {
    hosts => "localhost:9200"
    index => "nginx-%{+YYYY.MM.dd}"
  }
}
```
Проверка синтаксиса конфигов logstash
```
[root@elk ~]# /usr/share/logstash/bin/logstash -t -f /etc/logstash/conf.d/
```


#### Установка Filebeat для отправки логов в Logstash

Передем на `web` хост и установим и настроим

Добавим репозиторий
```
[root@web ~]# vi /etc/yum.repos.d/CentOS-ELK.repo

[elasticsearch-7.x]
name=ELK repository for 7.x packages
baseurl=https://artifacts.elastic.co/packages/7.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
```
```
[root@web ~]# yum install -y filebeat
```

Конфиг /etc/filebeat/filebeat.yml для отправки логов в logstash
```
[root@web ~]# vi /etc/filebeat/filebeat.yml
filebeat.inputs:
- type: log
  enabled: true
  paths:
      - /var/log/nginx
  fields:
    type: nginx_access
  fields_under_root: true
  scan_frequency: 5s
  
- type: log
  enabled: true
  paths:
      - /var/log/nginx
  fields:
    type: nginx_error
  fields_under_root: true
  scan_frequency: 5s

output.logstash:
  hosts: ["192.168.11.103:5044"]
```
```
[root@web ~]# systemctl enable --now filebeat
```

Проверка конфигурации lebeat:
```
[root@web ~]# filebeat test config
```

Ссылка на дополнительную информацию
- [Elasticsearch + Kibana + Logstash на CentOS](https://www.dmosk.ru/instruktions.php?object=elk-centos)
- [Как установить и настроить Elasticsearch, Logstash, Kibana](https://serveradmin.ru/ustanovka-i-nastroyka-elasticsearch-logstash-kibana-elk-stack/)

