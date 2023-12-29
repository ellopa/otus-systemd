## Systemd.

### 1.Написать сервис.

	Условия:
	- Раз в 30 секунд мониторить лог на предмет наличия ключевого слова.
	- Файл и слово должны задаваться в /etc/sysconfig.

- Создать файл с конфигурацией для сервиса в директории /etc/sysconfig - из неё сервис будет брать необходимые переменные /etc/sysconfig/watchlog

```bash
[root@dz8 ~]# cat /etc/sysconfig/watchlog
# Configuration file for my watchlog service
# Place it to /etc/sysconfig
# File and word in that file that we will be monit
WORD="ALERT"
LOG=/var/log/watchlog.log
```

- Создать /var/log/watchlog.log и со строками любыми и плюс ключевое слово **ALERT**
```
[root@dz8 ~]# cat /var/log/watchlog.log  
ALERT
QWE
123
gdfgfdg
gdfh
ALER
ht6fh�
change
ALERT
```

- Создать скрипт /opt/watchlog.sh, получилось 

```bash
[root@dz8 /]# cat /opt/watchlog.sh
#!/bin/bash
WORD=$1
LOG=$2
DATE=`date`
if grep $WORD $LOG &> /dev/null
then
logger "$DATE: I found word, Master!"
else
exit 0
fi
```
>- Команда logger отправляет лог в системный журнал

- Добавить права на запуск файла
```
[root@dz8 /]# chmod +x /opt/watchlog.sh
```

- Создать юнит для сервиса watchlog - /etc/systemd/system/watchlog.service

```
[root@dz8 /]# touch /etc/systemd/system/watchlog.service
[root@dz8 /]# nano /etc/systemd/system/watchlog.service
[root@dz8 /]# cat /etc/systemd/system/watchlog.service
[Unit]
Description=My watchlog service

[Service]
Type=oneshot
EnvironmentFile=/etc/sysconfig/watchlog
ExecStart=/opt/watchlog.sh $WORD $LOG
```

- Создать юнит для таймера - /etc/systemd/system/watchlog.timer

```
[root@dz8 /]# touch /etc/systemd/system/watchlog.timer
[root@dz8 /]# nano /etc/systemd/system/watchlog.timer
[root@dz8 /]# cat /etc/systemd/system/watchlog.timer
[Unit]
Description=Run watchlog script every 30 second

[Timer]
# Run every 30 second
OnUnitActiveSec=30
Unit=watchlog.service

[Install]
WantedBy=multi-user.target
```

- Стартануть и проверить

```
[root@dz8 /]# systemctl start watchlog.service
```
```
[root@dz8 /]# systemctl status watchlog.service
● watchlog.service - My watchlog service
   Loaded: loaded (/etc/systemd/system/watchlog.service; static; vendor preset: disabled)
   Active: activating (start) since Fri 2023-12-29 07:02:28 UTC; 1ms ago
 Main PID: 1297 (watchlog.sh)
   CGroup: /system.slice/watchlog.service
           └─1297 /bin/bash /opt/watchlog.sh ALERT /var/log/watchlog.log
Dec 29 07:02:28 dz8 systemd[1]: Starting My watchlog service...
```
```
[root@dz8 /]# systemctl start watchlog.timer
```
```bash

[root@dz8 /]# tail -f /var/log/messages
Dec 29 07:01:01 dz8 systemd: Started Session 2 of user root.
Dec 29 07:01:50 dz8 systemd: Starting My watchlog service...
Dec 29 07:01:50 dz8 root: Fri Dec 29 07:01:50 UTC 2023: I found word, Master!
Dec 29 07:01:50 dz8 systemd: Started My watchlog service.
Dec 29 07:02:28 dz8 systemd: Starting My watchlog service...
Dec 29 07:02:28 dz8 root: Fri Dec 29 07:02:28 UTC 2023: I found word, Master!

```

### 2. Из epel установим spawn-fcgi и перепишем init-скрипт на unit-файл. Имя сервиса должно также называться.

- Устанавливить spawn-fcgi и необходимые для него пакеты

```bash
[root@dz8 ~]# yum install epel-release -y && yum install spawn-fcgi php php-cli mod_fcgid httpd -y
Failed to set locale, defaulting to C
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
 * base: mirror.ams1.nl.leaseweb.net
 * extras: mirror.serverius.net
* updates: mirror.serverius.net
…………………….
Installed:
  httpd.x86_64 0:2.4.6-99.el7.centos.1   mod_fcgid.x86_64 0:2.3.9-6.el7   php.x86_64 0:5.4.16-48.el7   php-cli.x86_64 0:5.4.16-48.el7  
  spawn-fcgi.x86_64 0:1.6.3-5.el7       

Dependency Installed:
  apr.x86_64 0:1.4.8-7.el7                         apr-util.x86_64 0:1.5.2-6.el7_9.1       centos-logos.noarch 0:70.0.6-3.el7.centos      
  httpd-tools.x86_64 0:2.4.6-99.el7.centos.1       libzip.x86_64 0:0.10.1-8.el7            mailcap.noarch 0:2.1.41-2.el7                  
  php-common.x86_64 0:5.4.16-48.el7               

Complete!
```
> /etc/rc.d/init.d/spawn-fcg - cам Init скрипт, который будем переписывать, но перед этим необходимо раскомментировать строки с переменными в /etc/sysconfig/spawn-fcgi

```
[root@dz8 ~]# nano /etc/sysconfig/spawn-fcgi
[root@dz8 ~]# cat /etc/sysconfig/spawn-fcgi
# You must set some working options before the "spawn-fcgi" service will work.
# If SOCKET points to a file, then this file is cleaned up by the init script.
#
# See spawn-fcgi(1) for all possible options.
#
# Example :
SOCKET=/var/run/php-fcgi.sock
OPTIONS="-u apache -g apache -s $SOCKET -S -M 0600 -C 32 -F 1 -P /var/run/spawn-fcgi.pid -- /usr/bin/php-cgi"
```

- Создать unit-файл для spawn-fcgi сервиса
```
[root@dz8 ~]# nano /etc/systemd/system/spawn-fcgi.service
[root@dz8 ~]# cat /etc/systemd/system/spawn-fcgi.service
[Unit]
Description=Spawn-fcgi startup service by Otus
After=network.target

[Service]
Type=simple
PIDFile=/var/run/spawn-fcgi.pid
EnvironmentFile=/etc/sysconfig/spawn-fcgi
ExecStart=/usr/bin/spawn-fcgi -n $OPTIONS
KillMode=process

[Install]
WantedBy=multi-user.target
```
>- Запустить сервис spawn-fcgi.service
```
[root@dz8 ~]# systemctl daemon-reload
[root@dz8 ~]# systemctl enable --now spawn-fcgi.service
[root@dz8 ~]# systemctl start spawn-fcgi
```
- Проверить, что все успешно работает

```
[root@dz8 ~]# systemctl status spawn-fcgi
● spawn-fcgi.service - Spawn-fcgi startup service by Otus
   Loaded: loaded (/etc/systemd/system/spawn-fcgi.service; disabled; vendor preset: disabled)
   Active: active (running) since Fri 2023-12-29 08:40:57 UTC; 14s ago
 Main PID: 1181 (php-cgi)
   CGroup: /system.slice/spawn-fcgi.service
           ├─1181 /usr/bin/php-cgi
           ├─1182 /usr/bin/php-cgi
           ├─1183 /usr/bin/php-cgi
           ├─1184 /usr/bin/php-cgi
           ├─1185 /usr/bin/php-cgi
           ├─1186 /usr/bin/php-cgi
           ├─1187 /usr/bin/php-cgi
           ├─1188 /usr/bin/php-cgi
           ├─1189 /usr/bin/php-cgi
           ├─1190 /usr/bin/php-cgi
           ├─1191 /usr/bin/php-cgi
           ├─1192 /usr/bin/php-cgi
           ├─1193 /usr/bin/php-cgi
           ├─1194 /usr/bin/php-cgi
           ├─1195 /usr/bin/php-cgi
           ├─1196 /usr/bin/php-cgi
           ├─1197 /usr/bin/php-cgi
           ├─1198 /usr/bin/php-cgi
           ├─1199 /usr/bin/php-cgi
           ├─1200 /usr/bin/php-cgi
           ├─1201 /usr/bin/php-cgi
           ├─1202 /usr/bin/php-cgi
           ├─1203 /usr/bin/php-cgi
           ├─1204 /usr/bin/php-cgi
           ├─1205 /usr/bin/php-cgi
           ├─1206 /usr/bin/php-cgi
           ├─1207 /usr/bin/php-cgi
           ├─1208 /usr/bin/php-cgi
           ├─1209 /usr/bin/php-cgi
           ├─1210 /usr/bin/php-cgi
           ├─1211 /usr/bin/php-cgi
           ├─1212 /usr/bin/php-cgi
           └─1213 /usr/bin/php-cgi

Dec 29 08:40:57 dz8 systemd[1]: Started Spawn-fcgi startup service by Otus.
```

### 3. Дополнить юнит-файл apache httpd возможностью запустить несколько инстансов сервера с разными конфигами

- Установить софт (опционально) и настроить selinux
>- yum install nano -y
>- yum install -y policycoreutils-python
>- semanage port -m -t http_port_t -p tcp 8081
>- emanage port -m -t http_port_t -p tcp 8082
    
- Скопировать и изменить файл сервиса httpd.service в шаблон
>- cp /usr/lib/systemd/system/httpd.service /etc/systemd/system/httpd@.service
>- sed -i 's*EnvironmentFile=/etc/sysconfig/httpd*EnvironmentFile=/etc/sysconfig/%i*' /etc/systemd/system/httpd@.service
```
[root@dz8 ~]# cp /usr/lib/systemd/system/httpd.service /etc/systemd/system/httpd@.service
[root@dz8 ~]# sed -i 's*EnvironmentFile=/etc/sysconfig/httpd*EnvironmentFile=/etc/sysconfig/%i*' /etc/systemd/system/httpd@.service
```

- Скопируем и изменим файлы настройки сервиса httpd.service
>- cp /etc/sysconfig/httpd /etc/sysconfig/conf1
>- cp /etc/sysconfig/httpd /etc/sysconfig/conf2
>- sed -i 's*#OPTIONS=*OPTIONS=-f /etc/httpd/conf/httpd1.conf*' /etc/sysconfig/conf1
>- sed -i 's*#OPTIONS=*OPTIONS=-f /etc/httpd/conf/httpd2.conf*' /etc/sysconfig/conf2
    
```
[root@dz8 ~]# cp /etc/sysconfig/httpd /etc/sysconfig/conf1
[root@dz8 ~]# cp /etc/sysconfig/httpd /etc/sysconfig/conf2
[root@dz8 ~]# sed -i 's*#OPTIONS=*OPTIONS=-f /etc/httpd/conf/httpd1.conf*' /etc/sysconfig/conf1
[root@dz8 ~]# sed -i 's*#OPTIONS=*OPTIONS=-f /etc/httpd/conf/httpd2.conf*' /etc/sysconfig/conf2
```
```
[root@dz8 ~]# cat /etc/sysconfig/conf2
#
# This file can be used to set additional environment variables for
# the httpd process, or pass additional options to the httpd
# executable.
#
# Note: With previous versions of httpd, the MPM could be changed by
# editing an "HTTPD" variable here.  With the current version, that
# variable is now ignored.  The MPM is a loadable module, and the
# choice of MPM can be changed by editing the configuration file
# /etc/httpd/conf.modules.d/00-mpm.conf.
# 

#
# To pass additional options (for instance, -D definitions) to the
# httpd binary at startup, set OPTIONS here.
#
OPTIONS=-f /etc/httpd/conf/httpd2.conf

#
# This setting ensures the httpd process is started in the "C" locale
# by default.  (Some modules will not behave correctly if
# case-sensitive string comparisons are performed in a different
# locale.)
#
LANG=C
```
```
[root@dz8 ~]# cat /etc/sysconfig/conf1
#
# This file can be used to set additional environment variables for
# the httpd process, or pass additional options to the httpd
# executable.
#
# Note: With previous versions of httpd, the MPM could be changed by
# editing an "HTTPD" variable here.  With the current version, that
# variable is now ignored.  The MPM is a loadable module, and the
# choice of MPM can be changed by editing the configuration file
# /etc/httpd/conf.modules.d/00-mpm.conf.
# 

#
# To pass additional options (for instance, -D definitions) to the
# httpd binary at startup, set OPTIONS here.
#
OPTIONS=-f /etc/httpd/conf/httpd1.conf

#
# This setting ensures the httpd process is started in the "C" locale
# by default.  (Some modules will not behave correctly if
# case-sensitive string comparisons are performed in a different
# locale.)
#
LANG=C
```
- Скопировать и изменить файлы настройки демона httpd
>- cp /etc/httpd/conf/httpd.conf /etc/httpd/conf/httpd1.conf
>- cp /etc/httpd/conf/httpd.conf /etc/httpd/conf/httpd2.conf
>- используем разные порты
>- sed -i 's/Listen 80/Listen 8081/' /etc/httpd/conf/httpd1.conf
>- sed -i 's/Listen 80/Listen 8082/' /etc/httpd/conf/httpd2.conf
>- и разные pid файлы
>- echo "PidFile /var/run/httpd/httpd1.pid" >> /etc/httpd/conf/httpd1.conf
>- echo "PidFile /var/run/httpd/httpd2.pid" >> /etc/httpd/conf/httpd2.conf
```
[root@dz8 ~]# cp /etc/httpd/conf/httpd.conf /etc/httpd/conf/httpd1.conf
[root@dz8 ~]# cp /etc/httpd/conf/httpd.conf /etc/httpd/conf/httpd2.conf
[root@dz8 ~]# # используем разные порты
[root@dz8 ~]# sed -i 's/Listen 80/Listen 8081/' /etc/httpd/conf/httpd1.conf
[root@dz8 ~]# sed -i 's/Listen 80/Listen 8082/' /etc/httpd/conf/httpd2.conf
[root@dz8 ~]# # и разные pid файлы
[root@dz8 ~]# echo "PidFile /var/run/httpd/httpd1.pid" >> /etc/httpd/conf/httpd1.conf
[root@dz8 ~]# echo "PidFile /var/run/httpd/httpd2.pid" >> /etc/httpd/conf/httpd2.conf
```
```
[root@dz8 ~]# cat /etc/httpd/conf/httpd2.conf | grep Listen
# Listen: Allows you to bind Apache to specific IP addresses and/or
# Change this to Listen on specific IP addresses as shown below to 
#Listen 12.34.56.78:80
Listen 8082
[root@dz8 ~]# cat /etc/httpd/conf/httpd2.conf | grep Pid   
# least PidFile.
PidFile /var/run/httpd/httpd2.pid
```
```
[root@dz8 ~]# cat /etc/httpd/conf/httpd1.conf | grep Listen
# Listen: Allows you to bind Apache to specific IP addresses and/or
# Change this to Listen on specific IP addresses as shown below to 
#Listen 12.34.56.78:80
Listen 8081
[root@dz8 ~]# cat /etc/httpd/conf/httpd1.conf | grep Pid
# least PidFile.
PidFile /var/run/httpd/httpd1.pid
```
   
- Запустить два инстанса httpd
>- systemctl daemon-reload
>- systemctl enable --now httpd@conf1.service
>- systemctl enable --now httpd@conf2.service

```
[root@dz8 /]# systemctl enable --now httpd@conf1.service
Created symlink from /etc/systemd/system/multi-user.target.wants/httpd@conf1.service to /etc/systemd/system/httpd@.service.
[root@dz8SystemD /]# systemctl enable --now httpd@conf2.service
Created symlink from /etc/systemd/system/multi-user.target.wants/httpd@conf2.service to /etc/systemd/system/httpd@.service.
```
 
- Проверить статус сервисов
>- ```systemctl status httpd@conf1.service```
>- ```systemctl status httpd@conf2.service```
```
[root@dz8SystemD /]# systemctl status httpd@conf1.service
● httpd@conf1.service - The Apache HTTP Server
   Loaded: loaded (/etc/systemd/system/httpd@.service; enabled; vendor preset: disabled)
   Active: active (running) since Fri 2023-12-29 11:10:13 UTC; 53min ago
     Docs: man:httpd(8)
           man:apachectl(8)
 Main PID: 4416 (httpd)
   Status: "Total requests: 0; Current requests/sec: 0; Current traffic:   0 B/sec"
   CGroup: /system.slice/system-httpd.slice/httpd@conf1.service
           ├─4416 /usr/sbin/httpd -f /etc/httpd/conf/httpd1.conf -DFOREGROUND
           ├─4417 /usr/sbin/httpd -f /etc/httpd/conf/httpd1.conf -DFOREGROUND
           ├─4418 /usr/sbin/httpd -f /etc/httpd/conf/httpd1.conf -DFOREGROUND
           ├─4419 /usr/sbin/httpd -f /etc/httpd/conf/httpd1.conf -DFOREGROUND
           ├─4420 /usr/sbin/httpd -f /etc/httpd/conf/httpd1.conf -DFOREGROUND
           ├─4421 /usr/sbin/httpd -f /etc/httpd/conf/httpd1.conf -DFOREGROUND
           └─4422 /usr/sbin/httpd -f /etc/httpd/conf/httpd1.conf -DFOREGROUND

Dec 29 11:10:13 dz8SystemD systemd[1]: Starting The Apache HTTP Server...
Dec 29 11:10:13 dz8SystemD httpd[4416]: AH00558: httpd: Could not reliably determine the server's fully qualified domain name, u...message
Dec 29 11:10:13 dz8SystemD systemd[1]: Started The Apache HTTP Server.
Hint: Some lines were ellipsized, use -l to show in full.
```
```
root@dz8SystemD /]# systemctl status httpd@conf2.service
● httpd@conf2.service - The Apache HTTP Server
   Loaded: loaded (/etc/systemd/system/httpd@.service; enabled; vendor preset: disabled)
   Active: active (running) since Fri 2023-12-29 11:10:35 UTC; 54min ago
     Docs: man:httpd(8)
           man:apachectl(8)
 Main PID: 4444 (httpd)
   Status: "Total requests: 0; Current requests/sec: 0; Current traffic:   0 B/sec"
   CGroup: /system.slice/system-httpd.slice/httpd@conf2.service
           ├─4444 /usr/sbin/httpd -f /etc/httpd/conf/httpd2.conf -DFOREGROUND
           ├─4445 /usr/sbin/httpd -f /etc/httpd/conf/httpd2.conf -DFOREGROUND
           ├─4446 /usr/sbin/httpd -f /etc/httpd/conf/httpd2.conf -DFOREGROUND
           ├─4447 /usr/sbin/httpd -f /etc/httpd/conf/httpd2.conf -DFOREGROUND
           ├─4448 /usr/sbin/httpd -f /etc/httpd/conf/httpd2.conf -DFOREGROUND
           ├─4449 /usr/sbin/httpd -f /etc/httpd/conf/httpd2.conf -DFOREGROUND
           └─4450 /usr/sbin/httpd -f /etc/httpd/conf/httpd2.conf -DFOREGROUND

Dec 29 11:10:35 dz8SystemD systemd[1]: Starting The Apache HTTP Server...
Dec 29 11:10:35 dz8SystemD httpd[4444]: AH00558: httpd: Could not reliably determine the server's fully qualified domain name, u...message
Dec 29 11:10:35 dz8SystemD systemd[1]: Started The Apache HTTP Server.
Hint: Some lines were ellipsized, use -l to show in full.
```

>- Проверить можно несколькими способами, например посмотреть какие порты слушаются

```bash
[root@dz8 /]# ss -tnulp | grep httpd
tcp    LISTEN     0      128      :::8081                 :::*                   users:(("httpd",pid=4422,fd=4),("httpd",pid=4421,fd=4),("httpd",pid=4420,fd=4),("httpd",pid=4419,fd=4),("httpd",pid=4418,fd=4),("httpd",pid=4417,fd=4),("httpd",pid=4416,fd=4))
tcp    LISTEN     0      128      :::8082                 :::*                   users:(("httpd",pid=4450,fd=4),("httpd",pid=4449,fd=4),("httpd",pid=4448,fd=4),("httpd",pid=4447,fd=4),("httpd",pid=4446,fd=4),("httpd",pid=4445,fd=4),("httpd",pid=4444,fd=4))

```

