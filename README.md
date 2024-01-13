# Практика с SELinux
## Цель:
Тренируем умение работать с SELinux: диагностировать проблемы и модифицировать политики SELinux для корректной работы приложений, если это требуется.

### Задание 1 - Запустить nginx на нестандартном порту 3-мя разными способами.

Во время развёртывания стенда попытка запустить nginx завершится с ошибкой:
```
[vagrant@selinux ~]$ systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Sat 2024-01-13 11:55:54 UTC; 1min 13s ago
  Process: 2751 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
  Process: 2750 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)

```
Разрешим в SELinux работу nginx на порту TCP 4881 c помощью переключателей setsebool.\
\
Находим в логах (/var/log/audit/audit.log) информацию о блокировании порта, устанавливаем `policycoreutils-python` и с помощью утилиты `audit2why` смотрим почему трафик блокируется:
```
[root@selinux ~]# grep 1705146954.032:789 /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1705146954.032:789): avc:  denied  { name_bind } for  pid=2751 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0

	Was caused by:
	The boolean nis_enabled was set incorrectly. 
	Description:
	Allow nis to enabled

	Allow access by executing:
	# setsebool -P nis_enabled 1
```
Включим параметр nis_enabled и перезапустим nginx:
```
[root@selinux ~]# setsebool -P nis_enabled on
[root@selinux ~]# systemctl restart nginx
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Sat 2024-01-13 12:25:01 UTC; 10s ago
  Process: 21651 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 21649 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 21648 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 21653 (nginx)
   CGroup: /system.slice/nginx.service
           ├─21653 nginx: master process /usr/sbin/nginx
           └─21655 nginx: worker process

Jan 13 12:25:01 selinux systemd[1]: Starting The nginx HTTP and reverse pro.....
Jan 13 12:25:01 selinux nginx[21649]: nginx: the configuration file /etc/ng...ok
Jan 13 12:25:01 selinux nginx[21649]: nginx: configuration file /etc/nginx/...ul
Jan 13 12:25:01 selinux systemd[1]: Started The nginx HTTP and reverse prox...r.
Hint: Some lines were ellipsized, use -l to show in full.
```

Теперь разрешим в SELinux работу nginx на порту TCP 4881 c помощью добавления нестандартного порта в имеющийся тип:
```
[root@selinux ~]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
[root@selinux ~]# semanage port -a -t http_port_t -p tcp 4881
[root@selinux ~]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      4881, 80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
root@selinux ~]# systemctl restart nginx
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Sat 2024-01-13 12:30:46 UTC; 10s ago
  Process: 21696 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 21693 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 21692 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
```
Разрешим в SELinux работу nginx на порту TCP 4881 c помощью формирования и установки модуля SELinux:
```
[root@selinux ~]# systemctl restart nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
[root@selinux ~]# grep nginx /var/log/audit/audit.log | audit2allow -M nginx
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i nginx.pp
[root@selinux ~]# semodule -i nginx.pp
[root@selinux ~]# systemctl restart nginx
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Sat 2024-01-13 12:38:49 UTC; 5s ago
  Process: 21758 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 21756 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 21755 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
```
По моему, формирование и добавления модуля для nginx, более правильный способ управления правилами selinux.
### Задание 2 - Обеспечение работоспособности приложения при включенном SELinux
Для выполнения задания клонируем нужный репозиторий и запустим 2 VM.\
После этого подключимся к клиенту и попробуем внести изменения в зону:
```
[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
update failed: SERVFAIL
> quit
```
Смотрим ошибки:
```
[root@client ~]# cat /var/log/audit/audit.log | audit2why
[root@client ~]# 
```
Ошибок нет. Подключаемся к серверу и проверим там:
```
[root@ns01 ~]# cat /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1705151770.869:1889): avc:  denied  { create } for  pid=5024 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0

	Was caused by:
		Missing type enforcement (TE) allow rule.

		You can use audit2allow to generate a loadable module to allow this access.

```
Видно, что ошибка в контексте безопасности. Изменим контекст и попробуем внести изменения в зону ещё раз:
```
[root@ns01 ~]# sudo semanage fcontext -l | grep named
/etc/rndc.*                                        regular file       system_u:object_r:named_conf_t:s0 
/var/named(/.*)?                                   all files          system_u:object_r:named_zone_t:s0 
/etc/unbound(/.*)?                                 all files          system_u:object_r:named_conf_t:s0 
/var/run/bind(/.*)?                                all files          system_u:object_r:named_var_run_t:s0 
[root@ns01 ~]# sudo chcon -R -t named_zone_t /etc/named
[root@ns01 ~]# ls -laZ /etc/named
drw-rwx---. root named system_u:object_r:named_zone_t:s0 .
drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
drw-rwx---. root named unconfined_u:object_r:named_zone_t:s0 dynamic
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.50.168.192.rev
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab.view1
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.newdns.lab

[root@client ~]# nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
> quit
```
Всё заработало!\
![Скрин](https://github.com/FeeLinS9/lesson12/blob/master/img.png)\
Проблема была в том, что selinux блокировал доступ к файлам из-за неправильного контекста.