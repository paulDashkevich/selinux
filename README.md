# selinux - выполнение домашки
* 1. Запустить nginx на нестандартном порту 3-мя разными способами:
*переключатели setsebool;*
*добавление нестандартного порта в имеющийся тип;*
*формирование и установка модуля SELinux.*
Установка nginx и работа его на стандартных портах
```
yum install nginx
[root@centos ~]# systemctl status nginx.service -l
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Wed 2020-07-29 06:44:50 +03; 6s ago
  Process: 20363 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 20357 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 20351 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 20365 (nginx)
    Tasks: 5
   CGroup: /system.slice/nginx.service
           ├─20365 nginx: master process /usr/sbin/ngin
           ├─20366 nginx: worker proces
           ├─20367 nginx: worker proces
           ├─20368 nginx: worker proces
           └─20369 nginx: worker proces

Jul 29 06:44:50 centos.local systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jul 29 06:44:50 centos.local nginx[20357]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Jul 29 06:44:50 centos.local nginx[20357]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Jul 29 06:44:50 centos.local systemd[1]: Started The nginx HTTP and reverse proxy server.
```
меняем вручную порт nginx к примеру, 8092
```
...
 server {
        listen       8092 default_server;
```
проверяем и видим, что движок не работает и ругается на запрет старта
```
[root@centos ~]# systemctl start nginx.service -l
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
```
и в добавок вывод журнала
```
[root@centos ~]# systemctl status nginx.service
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Wed 2020-07-29 06:47:09 +03; 45s ago
  Process: 20428 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
  Process: 20423 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
Jul 29 06:47:09 centos.local systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jul 29 06:47:09 centos.local nginx[20428]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Jul 29 06:47:09 centos.local nginx[20428]: nginx: [emerg] bind() to 0.0.0.0:8092 failed (13: Permission denied)
Jul 29 06:47:09 centos.local nginx[20428]: nginx: configuration file /etc/nginx/nginx.conf test failed
Jul 29 06:47:09 centos.local systemd[1]: nginx.service: control process exited, code=exited status=1
Jul 29 06:47:09 centos.local systemd[1]: Failed to start The nginx HTTP and reverse proxy server.
Jul 29 06:47:09 centos.local systemd[1]: Unit nginx.service entered failed state.
Jul 29 06:47:09 centos.local systemd[1]: nginx.service failed.
```
первая проверка - отключаем SElinux и видим, что nginx стартует на нестандартном порту
# setenforce 0
```
[root@centos ~]# systemctl start nginx.service -l
[root@centos ~]# systemctl status nginx.service
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Wed 2020-07-29 06:49:33 +03; 24s ago
```
и ещё вывод команды ss -tulpn | grep nginx
```
[root@centos ~]# ss -tulpn | grep nginx
tcp    LISTEN     0      128       *:8092                  *:*                   users:(("nginx",pid=20540,fd=6),("nginx",pid=20539,fd=6),("nginx",pid=20538,fd=6),("nginx",pid=20537,fd=6),("nginx",pid=20536,fd=6))
```
Возвращаем SElinux на место и работаем уже по ДЗ
# setenforce 1
Для запуска движка есть несколько способов:
  *а - просмотреть утилитой audit2why < /var/log/audit/audit.log лог аудита и в выводе команды будет содержаться решение
 ```
 [root@centos ~]# audit2why < /var/log/audit/audit.log
type=AVC msg=audit(1595151161.047:334): avc:  denied  { read write } for  pid=21179 comm="nginx" name="nginx.pid" dev="tmpfs" ino=56767 scontext=system_u:system_r:httpd_t:s0 tcontext=unconfined_u:object_r:var_run_t:s0 tclass=file permissive=0
        Was caused by:
                Missing type enforcement (TE) allow rule.

                You can use audit2allow to generate a loadable module to allow this access.
type=AVC msg=audit(1595153625.769:409): avc:  denied  { search } for  pid=7270 comm="geoclue" name="7413" dev="proc" ino=71053 scontext=system_u:system_r:geoclue_t:s0 tcontext=system_u:system_r:unconfined_service_t:s0 tclass=dir permissive=0
        Was caused by:
                Missing type enforcement (TE) allow rule.

                You can use audit2allow to generate a loadable module to allow this access.
type=USER_AVC msg=audit(1595153626.603:411): pid=1086 uid=81 auid=4294967295 ses=4294967295 subj=system_u:system_r:system_dbusd_t:s0-s0:c0.c1023 msg='avc:  denied  { send_msg } for msgtype=method_call interface=org.freedesktop.DBus.Properties member=GetAll dest=:1.39 spid=7178 tpid=2412 scontext=system_u:system_r:unconfined_service_t:s0 tcontext=system_u:system_r:boltd_t:s0 tclass=dbus  exe="/usr/bin/dbus-daemon" sauid=81 hostname=? addr=? terminal=?'
        Was caused by:
                Missing type enforcement (TE) allow rule.
                You can use audit2allow to generate a loadable module to allow this access.
type=AVC msg=audit(1595153626.669:412): avc:  denied  { search } for  pid=7270 comm="geoclue" name="7178" dev="proc" ino=67443 scontext=system_u:system_r:geoclue_t:s0 tcontext=system_u:system_r:unconfined_service_t:s0 tclass=dir permissive=0
        Was caused by:
                Missing type enforcement (TE) allow rule.
                You can use audit2allow to generate a loadable module to allow this access.
type=AVC msg=audit(1595153823.588:184): avc:  denied  { search } for  pid=2874 comm="geoclue" name="3028" dev="proc" ino=36585 scontext=system_u:system_r:geoclue_t:s0 tcontext=system_u:system_r:unconfined_service_t:s0 tclass=dir permissive=0
        Was caused by:
                Missing type enforcement (TE) allow rule.
                You can use audit2allow to generate a loadable module to allow this access.
type=USER_AVC msg=audit(1595153824.314:187): pid=1109 uid=81 auid=4294967295 ses=4294967295 subj=system_u:system_r:system_dbusd_t:s0-s0:c0.c1023 msg='avc:  denied  { send_msg } for msgtype=method_call interface=org.freedesktop.DBus.Properties member=GetAll dest=:1.39 spid=2780 tpid=2367 scontext=system_u:system_r:unconfined_service_t:s0 tcontext=system_u:system_r:boltd_t:s0 tclass=dbus  exe="/usr/bin/dbus-daemon" sauid=81 hostname=? addr=? terminal=?'
        Was caused by:
                Missing type enforcement (TE) allow rule.
                You can use audit2allow to generate a loadable module to allow this access.
type=AVC msg=audit(1595153824.475:188): avc:  denied  { search } for  pid=2874 comm="geoclue" name="2780" dev="proc" ino=31615 scontext=system_u:system_r:geoclue_t:s0 tcontext=system_u:system_r:unconfined_service_t:s0 tclass=dir permissive=0
        Was caused by:
                Missing type enforcement (TE) allow rule.
                You can use audit2allow to generate a loadable module to allow this access.
type=AVC msg=audit(1595600438.850:6888): avc:  denied  { setcap } for  pid=6469 comm="sshd" scontext=system_u:system_r:sshd_t:s0-s0:c0.c1023 tcontext=system_u:system_r:sshd_t:s0-s0:c0.c1023 tclass=process permissive=0
        Was caused by:
                Missing type enforcement (TE) allow rule.
                You can use audit2allow to generate a loadable module to allow this access.
type=AVC msg=audit(1595600439.083:6896): avc:  denied  { setcap } for  pid=6478 comm="sshd" scontext=system_u:system_r:sshd_t:s0-s0:c0.c1023 tcontext=system_u:system_r:sshd_t:s0-s0:c0.c1023 tclass=process permissive=0
        Was caused by:
                Missing type enforcement (TE) allow rule.
                You can use audit2allow to generate a loadable module to allow this access.
type=AVC msg=audit(1595930667.937:249): avc:  denied  { search } for  pid=3877 comm="geoclue" name="4021" dev="proc" ino=48184 scontext=system_u:system_r:geoclue_t:s0 tcontext=system_u:system_r:unconfined_service_t:s0 tclass=dir permissive=0
        Was caused by:
                Missing type enforcement (TE) allow rule.
                You can use audit2allow to generate a loadable module to allow this access.
type=USER_AVC msg=audit(1595930668.609:251): pid=1082 uid=81 auid=4294967295 ses=4294967295 subj=system_u:system_r:system_dbusd_t:s0-s0:c0.c1023 msg='avc:  denied  { send_msg } for msgtype=method_call interface=org.freedesktop.DBus.Properties member=GetAll dest=:1.39 spid=3786 tpid=2404 scontext=system_u:system_r:unconfined_service_t:s0 tcontext=system_u:system_r:boltd_t:s0 tclass=dbus  exe="/usr/bin/dbus-daemon" sauid=81 hostname=? addr=? terminal=?'
        Was caused by:
                Missing type enforcement (TE) allow rule.
                You can use audit2allow to generate a loadable module to allow this access.
type=AVC msg=audit(1595930668.693:252): avc:  denied  { search } for  pid=3877 comm="geoclue" name="3786" dev="proc" ino=42993 scontext=system_u:system_r:geoclue_t:s0 tcontext=system_u:system_r:unconfined_service_t:s0 tclass=dir permissive=0
        Was caused by:
                Missing type enforcement (TE) allow rule.
                You can use audit2allow to generate a loadable module to allow this access.
type=AVC msg=audit(1595930747.642:257): avc:  denied  { name_bind } for  pid=4974 comm="nginx" src=8098 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
        Was caused by:
        The boolean nis_enabled was set incorrectly.
        Description:
        Allow nis to enabled
        Allow access by executing:
        # setsebool -P nis_enabled 1
type=AVC msg=audit(1595993427.772:1170): avc:  denied  { name_bind } for  pid=19928 comm="nginx" src=8098 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=1
        Was caused by:
        The boolean nis_enabled was set incorrectly.
        Description:
        Allow nis to enabled
        Allow access by executing:
        # setsebool -P nis_enabled 1
type=AVC msg=audit(1595993550.535:1175): avc:  denied  { name_bind } for  pid=20042 comm="nginx" src=8098 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
        Was caused by:
        The boolean nis_enabled was set incorrectly.
        Description:
        Allow nis to enabled
        Allow access by executing:
        # setsebool -P nis_enabled 1
type=AVC msg=audit(1595994429.633:1193): avc:  denied  { name_bind } for  pid=20428 comm="nginx" src=8092 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
        Was caused by:
        The boolean nis_enabled was set incorrectly.
        Description:
        Allow nis to enabled
        Allow access by executing:
        # setsebool -P nis_enabled 1
type=AVC msg=audit(1595994566.386:1195): avc:  denied  { name_bind } for  pid=20488 comm="nginx" src=8092 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
        Was caused by:
        The boolean nis_enabled was set incorrectly.
        Description:
        Allow nis to enabled
        Allow access by executing:
        # setsebool -P nis_enabled 1
type=AVC msg=audit(1595994573.481:1199): avc:  denied  { name_bind } for  pid=20528 comm="nginx" src=8092 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=1
        Was caused by:
        The boolean nis_enabled was set incorrectly.
        Description:
        Allow nis to enabled
        Allow access by executing:
        # setsebool -P nis_enabled 1
 ```
но такое решение позволит всем приложениям снять ограничения по безопасности, также видно следующее решение в этом выводе:
 *б - создать с помощью утилиты audit2allow загружаемый модуль по каждому случаю
 *в - переключатели boolean типа
Нужно узнать контекст безопасности для nginx
 ```
 ps aux -Z | grep nginx
 [root@centos ~]# ps aux | grep nginx
root     20536  0.0  0.0 120900  2244 ?        Ss   06:49   0:00 nginx: master process /usr/sbin/nginx
nginx    20537  0.0  0.0 123380  3520 ?        S    06:49   0:00 nginx: worker process
```
смотрим по pid контекст безопасности процесса:
```
[root@centos ~]# ps -Z 20537
LABEL                             PID TTY      STAT   TIME COMMAND
system_u:system_r:httpd_t:s0    20537 ?        S      0:00 nginx: worker process
```
работаем дальше с типом **httpd_t**
Смотрим разрешения запуска с портами
```
[root@centos ~]# semanage  port -l | grep http_port_t
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
```
Нашего порта, конечно, здесь нет. 
по указанным выше пунктам, можно разрешить порт. Используем **semanage**, чтобы добавить нестандартый порт в домен nginx
```
semanage port -a -t http_port_t -p tcp 8092
[root@centos ~]# systemctl status nginx.service
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Wed 2020-07-29 06:49:33 +03; 1h 13min ago
```
Следущий способ. sealert -a /var/log/audit/audit.log, где видим, что разрешить вопрос можно с помощью audit2allow
```
[root@centos ~]# sealert -a /var/log/audit/audit.log
100% done
found 4 alerts in /var/log/audit/audit.log
--------------------------------------------------------------------------------
SELinux is preventing /usr/sbin/nginx from 'read, write' accesses on the file nginx.pid.
allow this access for now by executing:
# ausearch -c 'nginx' --raw | audit2allow -M my-nginx
# semodule -i my-nginx.pp
```
Запускаем ausearch -c 'nginx' --raw | audit2allow -M my-nginx
```
[root@centos ~]# ausearch -c 'nginx' --raw | audit2allow -M my-nginx
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i my-nginx.pp
```
И выполняем **semodule -i my-nginx.pp**.
```
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Wed 2020-07-29 08:46:29 +03; 8s ago
```
Ломаем обратно nginx, удалив установленный модуль:
```
[root@centos nginx]# semodule -r my-nginx
libsemanage.semanage_direct_remove_key: Removing last my-nginx module (no other my-nginx module exists at another priority).
```
и, наконец, используя параметризованные политики (булевые) **setsebool -P nis_enabled 1** можно включить nginx.
## 2. Починить сервис 
https://github.com/mbfx/otus-linux-adm/tree/master/selinux_dns_problems
