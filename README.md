# Работа с системой SELinux.
## Запустить nginx на нестандартном порту 3-мя разными способами:

Для начала запусти стенд из методички для выполнения первой части ДЗ

```
sudo vagrant up selinux
```
и подключаемся к VM

```
sudo vagrant up
```
Убедимся, что в развернутом стенде NGINX установден, но не может быт запущен

```
[vagrant@selinux ~]$ systemctl status firewalld
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; disabled; vendor preset: enabled)
   Active: inactive (dead)
     Docs: man:firewalld(1)
[vagrant@selinux ~]$ nginx -t
nginx: [alert] could not open error log file: open() "/var/log/nginx/error.log" failed (13: Permission denied)
2023/03/06 12:35:00 [warn] 21606#21606: the "user" directive makes sense only if the master process runs with super-user privileges, ignored in /etc/nginx/nginx.conf:5
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
2023/03/06 12:35:00 [emerg] 21606#21606: open() "/run/nginx.pid" failed (13: Permission denied)
nginx: configuration file /etc/nginx/nginx.conf test failed
```
### Переключатели setsebool

Для начала поставим пкет `policycoreutils-python` для пользования утилитой `audit2why` 

```
yum install policycoreutils-python
```
Воспользовавшись утилитой `audit2why` обрабатываем сообщения аудита SELinux

```
[root@selinux ~]# audit2why < /var/log/audit/audit.log
type=AVC msg=audit(1678103850.014:780): avc:  denied  { name_bind } for  pid=2761 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0

	Was caused by:
	The boolean nis_enabled was set incorrectly. <--- причина сбоя
	Description:
	Allow nis to enabled

	Allow access by executing:
	# setsebool -P nis_enabled 1  <--- вариант устранения

```
Чем мы и воспользуемся

```
setsebool -P nis_enabled 1
```
Проверояем работоспособность NGINX

```
[root@selinux ~]# systemctl start nginx
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Mon 2023-03-06 13:16:56 UTC; 6s ago
  Process: 21917 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 21915 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 21914 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 21919 (nginx)
   CGroup: /system.slice/nginx.service
           ├─21919 nginx: master process /usr/sbin/nginx
           └─21921 nginx: worker process

Mar 06 13:16:56 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Mar 06 13:16:56 selinux nginx[21915]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Mar 06 13:16:56 selinux nginx[21915]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Mar 06 13:16:56 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```
Стучимся на страницу

```
[root@selinux ~]# curl -I 127.0.0.1:4881
HTTP/1.1 200 OK
Server: nginx/1.20.1
Date: Mon, 06 Mar 2023 13:18:27 GMT
Content-Type: text/html
Content-Length: 4833
Last-Modified: Fri, 16 May 2014 15:12:48 GMT
Connection: keep-alive
ETag: "53762af0-12e1"
Accept-Ranges: bytes
```
Возвращаем все в исходное состояние

```
[root@selinux ~]# setsebool -P nis_enabled 0
```

### Добавление нестандартного порта в имеющийся тип.

Поменяем порт прослушки

```
[root@selinux ~]# cat  /etc/nginx/nginx.conf | grep 4881
        listen       4881;
        listen       [::]:4881;
[root@selinux ~]# sed -i s@4881@8087@g /etc/nginx/nginx.conf
[root@selinux ~]# cat  /etc/nginx/nginx.conf | grep 808 
        listen       8087;
        listen       [::]:8087;
```

Смотрим утилитой `semanage` дефолтный порты http трафика

```
[root@selinux ~]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
```
Добавляем этой же утилитой наш нестандартный порт

```
[root@selinux ~]# semanage port -a -t http_port_t -p tcp 8087
```
еще раз проверяем порты http трафика

```
[root@selinux ~]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      8087, 80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
```
Стартуем NGINX

```
[root@selinux ~]# systemctl start nginx
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Mon 2023-03-06 13:56:59 UTC; 7s ago
  Process: 22202 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 22200 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 22199 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 22204 (nginx)
   CGroup: /system.slice/nginx.service
           ├─22204 nginx: master process /usr/sbin/nginx
           └─22205 nginx: worker process

Mar 06 13:56:59 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Mar 06 13:56:59 selinux nginx[22200]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Mar 06 13:56:59 selinux nginx[22200]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Mar 06 13:56:59 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```

Стучимся на NGINX

```
[root@selinux ~]# curl -I 127.0.0.1:8087
HTTP/1.1 200 OK
Server: nginx/1.20.1
Date: Mon, 06 Mar 2023 14:08:59 GMT
Content-Type: text/html
Content-Length: 4833
Last-Modified: Fri, 16 May 2014 15:12:48 GMT
Connection: keep-alive
ETag: "53762af0-12e1"
Accept-Ranges: bytes
```
Возвращаем все в исходное состояние

```
[root@selinux ~]# semanage port -d -t http_port_t -p tcp 8087
```

### Формирование и установка модуля SELinux.

Еще раз поменяем порт прослушки и убедимся, что NGINX на нем не стартует

```
[root@selinux ~]# sed -i s@8087@8088@g /etc/nginx/nginx.conf
[root@selinux ~]# cat  /etc/nginx/nginx.conf | grep 808
        listen       8088;
        listen       [::]:8088;
[root@selinux ~]# systemctl start nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
```

Делаем `grep` файла `/var/log/audit/audit.log` по nginx, смотрим ошибки

```
grep nginx /var/log/audit/audit.log
```
Смотрим длинный вывод и при помощи утилиты `audit2allow` формируем разрешающий модуль SELinux

```
[root@selinux ~]# grep nginx /var/log/audit/audit.log | audit2allow -M nginx
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i nginx.pp
```

задействуем его

```
semodule -i nginx.pp
```

Стартуем NGINX и убеждаемся в его работоспособности

```
[root@selinux ~]# systemctl start nginx
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Mon 2023-03-06 14:34:36 UTC; 7s ago
  Process: 22507 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 22505 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 22504 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 22509 (nginx)
   CGroup: /system.slice/nginx.service
           ├─22509 nginx: master process /usr/sbin/nginx
           └─22511 nginx: worker process
[root@selinux ~]# curl -I 127.0.0.1:8088
HTTP/1.1 200 OK
Server: nginx/1.20.1
Date: Mon, 06 Mar 2023 14:43:11 GMT
Content-Type: text/html
Content-Length: 4833
Last-Modified: Fri, 16 May 2014 15:12:48 GMT
Connection: keep-alive
ETag: "53762af0-12e1"
Accept-Ranges: bytes
```
Можно откатиться назад

```
semodule -r nginx
```

 ВАЖНО ПОМНИТЬ, ЧТО В ПЕРВОМ И ВТОРОМ МОДУЛИ СОХРАНЯЮТ СВОЮ РАБОТОСПОСОБНОСТЬ ДО ПЕРЕЗАГРУЗКИ СЕРВРЕА!!!

## Обеспечить работоспособность приложения при включенном SELinux.

Разворачиваем стенд разработанный для отработки ДЗ.

Схема стенда следующая:

```
- ns01 - DNS-сервер (192.168.50.10);

- client - клиентская рабочая станция (192.168.50.15).
```
Заходим на `clien `

```
roman@root-ubuntu:~/selinux/git/otus-linux-adm/selinux_dns_problems$ sudo vagrant ssh client
Last login: Tue Mar  7 08:17:29 2023 from 10.0.2.2
###############################
### Welcome to the DNS lab! ###
###############################

- Use this client to test the enviroment
- with dig or nslookup. Ex:
    dig @192.168.50.10 ns01.dns.lab

- nsupdate is available in the ddns.lab zone. Ex:
    nsupdate -k /etc/named.zonetransfer.key
    server 192.168.50.10
    zone ddns.lab 
    update add www.ddns.lab. 60 A 192.168.50.15
    send

- rndc is also available to manage the servers
    rndc -c ~/rndc.conf reload

###############################
### Enjoy! ####################
###############################
```
Выполняем указанные в приветствии VM команды, тем самым пытаемся удаленно (с рабочей станции) внести изменения в зону ddns.lab

```
[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
update failed: SERVFAIL
> 
```
Получаем ошибку.

Смотрим на `clien ` утилитой `audit2why` ошибки

```
[root@client ~]# cat /var/log/audit/audit.log | audit2why
[root@client ~]# 
```
здесь все чисто, идем на `ns01`, повторяем запрос

```
[root@ns01 ~]# cat /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1678178094.596:1869): avc:  denied  { create } for  pid=4996 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0

	Was caused by:
		Missing type enforcement (TE) allow rule.

		You can use audit2allow to generate a loadable module to allow this access.
```
После вывода утилиты я решил по аналогии с первой частью ДЗ воспользоваться утилитой `audit2allow` и сформировать разрешающий модуль SELinux. 
Применение сгенерированного  модуля не увенчалось успехом.

Следующим шагом стало ознакомление с дефолтными правилами для `named`

```
semanage fcontext --list | grep named
```
Проанализировав вывод заключаем, что все паравила для `named` имеют контекст `named_t`, а не `etc_t` о чем и говорит утилита `audit2why`.

Собственно это и наблюдаем в каталоге `/etc/named`

```
[root@ns01 dynamic]# ls -laZ /etc/named                      
drw-rwx---. root named system_u:object_r:etc_t:s0       .
drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
drw-rwx---. root named unconfined_u:object_r:etc_t:s0   dynamic
-rw-rw----. root named system_u:object_r:etc_t:s0       named.50.168.192.rev
-rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab
-rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab.view1
-rw-rw----. root named system_u:object_r:etc_t:s0       named.newdns.lab
```
Воспользовавшись методичкой и применив команду

```
sudo chcon -R -t named_zone_t /etc/named
```
все встало на свои места
```
[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
> quit
[vagrant@client ~]$ dig www.ddns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.13 <<>> www.ddns.lab
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 6572
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.ddns.lab.			IN	A

;; ANSWER SECTION:
www.ddns.lab.		60	IN	A	192.168.50.15

;; AUTHORITY SECTION:
ddns.lab.		3600	IN	NS	ns01.dns.lab.

;; ADDITIONAL SECTION:
ns01.dns.lab.		3600	IN	A	192.168.50.10

;; Query time: 1 msec
;; SERVER: 192.168.50.10#53(192.168.50.10)
;; WHEN: Tue Mar 07 14:34:46 UTC 2023
;; MSG SIZE  rcvd: 96

```
### P.S.

 Была еще неудачная попытка применить права только к файлу журанла указанному в выводе утилиты `audit2allow`

```
sudo chcon -R -t named_zone_t /etc/named/dynamic/named.ddns.lab.view1.jnl
```
Еще одним из возможных решений этой задачи м.б. перенос всей инфраструктуры развернутой `playbook` из учебного стенда из дирректории `/etc/named` в дефолтную `/var/named`.