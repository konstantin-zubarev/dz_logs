### 1. Web

#### 1.1. Установить Nginx

Для установки Nginx используем репозиторийт EPEL
```
[root@web ~]# yum install -y epel-release
```
Затем можно установить Nginx:
```
[root@web ~]# yum install -y nginx
```
Необходимо добавить программу в автозагрузку и запустить сервис Nginx.
```
[root@web ~]# systemctl enable --now nginx
```
Отредактирум файл `nginx.conf` и добавим строки
```
[root@web ~]# vi /etc/nginx/nginx.conf
```
```
error_log /var/log/nginx/error.log crit;
error_log syslog:server={{ nginx['log_server'] }}:{{ nginx['log_server_port'] }},facility=local7,tag=nginx_error,severity=info;
pid /run/nginx.pid;


access_log syslog:server={{ nginx['log_server'] }}:{{ nginx['log_server_port'] }},facility=local7,tag=nginx_access,severity=info main;
```
Перезагрузить Nginx
```
[root@web ~]# systemctl restart nginx.service
```

#### 1.2. Настроить rsyslog для сервира

Изменить в файле `rsyslog.conf`
```
[root@web ~]# vi /etc/rsyslog.conf
```
Добавить строчку `*.crit @@192.168.11.101:514`

Перезапустить сервис Rsyslog.
```
[root@web ~]# systemctl restart rsyslog.service
```

#### 1.3. Установить плагин и настроить аудит на изменение конфигов nginx
```
[root@web ~]# yum install -y audispd-plugins
```
Изменить опции конфиг файла:
- в файле `/etc/audisp/audisp-remote.conf`
```
	remote_server = 192.168.11.101
	#port = 514
```
- в файле `/etc/audisp/plugins.d/au-remote.conf`
```
	active = yes
```
- в файле `/etc/audit/auditd.conf`
```
	write_logs = no
```
Добавим правило, по которому можно будет отслеживать события.
```
[root@web ~]# auditctl -w /etc/nginx/ -p wa -k nginx_conf
[root@web ~]# auditctl -l >> /etc/audit/rules.d/nginx_conf.rules
```
Перезагрузить auditd
```
[root@web ~]# service auditd restart
```
Проверить правило
```
[root@web ~]# auditctl -l
```

### 2. Log

#### 2.1. Настроить Rsyslog на сервере

Изменить в файле `rsyslog.conf` для приемов логов
```
[root@log ~]# vi /etc/rsyslog.conf
```
```
	# Provides UDP syslog reception
	$ModLoad imudp
	$UDPServerRun 514

	# Provides TCP syslog reception
	$ModLoad imtcp
	$InputTCPServerRun 514
	
	$template RemoteLogs,"/var/log/rsyslog/%HOSTNAME%/%PROGRAMNAME%.log"
	*.* ?RemoteLogs
	& ~
```
Перезапустить сервис Rsyslog.
```
[rroot@log ~]# systemctl restart rsyslog.service
```
#### 2.2. Установить плагин `audispd-plugins`

```
[root@log ~]# yum install -y audispd-plugins
```

Раскоментировать строчку в файле `/etc/audit/auditd.conf`
```
	tcp_listen_port = 60
```

Перезагрузить auditd
```
	[root@log ~]# service auditd restart
```

Ссылка на дополнительную информацию
- [Настройка rsyslog для хранения логов на удаленном сервере](https://www.dmosk.ru/miniinstruktions.php?mini=rsyslog#install)
