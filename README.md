# Lesson15_SELinux

## Задание1
Запустить nginx на нестандартном порту 3-мя разными способами:
- переключатели setsebool;
- добавление нестандартного порта в имеющийся тип;
- формирование и установка модуля SELinux.


## Выполнение

останавливаем и проверяем что firewall выключен
```
[root@localhost sadmin]# systemctl stop firewalld.service
[root@localhost sadmin]# systemctl status firewalld.service
WARNING: terminal is not fully functional
Press RETURN to continue
○ firewalld.service - firewalld - dynamic firewall daemon
     Loaded: loaded (/usr/lib/systemd/system/firewalld.service; enabled; pre>
     Active: inactive (dead) since Sun 2026-05-31 15:15:43 MSK; 10s ago
   Duration: 3w 6d 23h 13min 31.978s
       Docs: man:firewalld(1)
    Process: 788 ExecStart=/usr/sbin/firewalld --nofork --nopid $FIREWALLD_A>
   Main PID: 788 (code=exited, status=0/SUCCESS)
        CPU: 1.017s

мая 03 16:01:43 localhost systemd[1]: Starting firewalld - dynamic firewall >
мая 03 16:01:56 localhost systemd[1]: Started firewalld - dynamic firewall d>
мая 31 15:15:28 localhost.localdomain systemd[1]: Stopping firewalld - dynam>
мая 31 15:15:43 localhost.localdomain systemd[1]: firewalld.service: Deactiv>
мая 31 15:15:43 localhost.localdomain systemd[1]: Stopped firewalld - dynami>
мая 31 15:15:43 localhost.localdomain systemd[1]: firewalld.service: Consume>

```

Vagrand и VirrtualBox не использую, ставим nginx вручную
```
dnf install nginx

```

проверяем режим работы selinux
```
[root@localhost sadmin]# getenforce
Enforcing
[root@localhost sadmin]#
```

меняем порт у nginx на 93, перезапускаем и убеждаемся, что не стартует

```
 server {
        index index.html index.htm;
        autoindex on;
        listen       93;
        listen       [::]:93;
        server_name  _;
        root         /usr/share/nginx/html;


[root@localhost sadmin]# systemctl restart nginx.service
Job for nginx.service failed because the control process exited with error code.
See "systemctl status nginx.service" and "journalctl -xeu nginx.service" for details.


[root@localhost sadmin]# systemctl status nginx.service
× nginx.service - The nginx HTTP and reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; pres>
     Active: failed (Result: exit-code) since Sun 2026-05-31 15:41:05 MSK;>
   Duration: 3w 6d 20h 31min 20.931s
    Process: 231811 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exite>
    Process: 231812 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1>
        CPU: 31ms

мая 31 15:41:04 localhost.localdomain systemd[1]: Starting The nginx HTTP >
мая 31 15:41:05 localhost.localdomain nginx[231812]: nginx: the configurat>
мая 31 15:41:05 localhost.localdomain nginx[231812]: nginx: [emerg] bind()>
мая 31 15:41:05 localhost.localdomain nginx[231812]: nginx: configuration >
мая 31 15:41:05 localhost.localdomain systemd[1]: nginx.service: Control p>
мая 31 15:41:05 localhost.localdomain systemd[1]: nginx.service: Failed wi>
мая 31 15:41:05 localhost.localdomain systemd[1]: Failed to start The ngin>


```

выполняем semanage port -a -t http_port_t -p tcp 93 и проверяем что nginx запустился таки на 93 порту

```
semanage port -a -t http_port_t -p tcp 93
root@localhost nginx]# systemctl restart nginx
[root@localhost nginx]# ss -tlnp | grep nginx
LISTEN 0      511          0.0.0.0:93        0.0.0.0:*    users:(("nginx",pid=232019,fd=6),("nginx",pid=232018,fd=6))
LISTEN 0      511             [::]:93           [::]:*    users:(("nginx",pid=232019,fd=7),("nginx",pid=232018,fd=7))
```

снова удаляем порт 93, перезапускаем nginx, видим что работать перестало.
```
[root@localhost nginx]# semanage port -d -t http_port_t -p tcp 93
[root@localhost nginx]# systemctl restart nginx
Job for nginx.service failed because the control process exited with error code.
See "systemctl status nginx.service" and "journalctl -xeu nginx.service" for details.
```

Смотрим в лог причину того что не запустилось
```
[root@localhost nginx]# grep 93  /var/log/audit/audit.log | audit2why

# вывод
ype=AVC msg=audit(1780233621.467:3596): avc:  denied  { name_bind } for  pid=232046 comm="nginx" src=93 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:reserved_port_t:s0 tclass=tcp_socket permissive=0

        Was caused by:
                Missing type enforcement (TE) allow rule.

                You can use audit2allow to generate a loadable module to allow this access.

```

Так как порт 93 не высокий, audid2why не покажет по нему информацию. поменяем порт на 4093

```
и теперь видим всю информацию
[root@localhost nginx]# grep 4093  /var/log/audit/audit.log | audit2why

root@localhost nginx]# grep 4093  /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1780234789.327:3617): avc:  denied  { name_bind } for  pid=232345 comm="nginx" src=4093 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0

        Was caused by:
        The boolean nis_enabled was set incorrectly.
        Description:
        Allow nis to enabled

        Allow access by executing:
        # setsebool -P nis_enabled 1
```

включаем параметр nis_enabled и проверяем
```
[root@localhost nginx]# setsebool -P nis_enabled on
[root@localhost nginx]# systemctl restart nginx
[root@localhost nginx]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; preset: disabl>
     Active: active (running) since Sun 2026-05-31 16:44:17 MSK; 14s ago
    Process: 232373 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=>
    Process: 232374 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
    Process: 232375 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
   Main PID: 232376 (nginx)
      Tasks: 2 (limit: 10464)
     Memory: 3.9M (peak: 4.1M)
        CPU: 36ms
     CGroup: /system.slice/nginx.service
             ├─232376 "nginx: master process /usr/sbin/nginx"
             └─232377 "nginx: worker process"

[root@localhost sadmin]# ss -tlnp | grep nginx
LISTEN 0      511          0.0.0.0:4093      0.0.0.0:*    users:(("nginx",pid=232377,fd=6),("nginx",pid=232376,fd=6))
LISTEN 0      511             [::]:4093         [::]:*    users:(("nginx",pid=232377,fd=7),("nginx",pid=232376,fd=7))
[root@localhost sadmin]#
```

отключим обратно 
```
setsebool -P nis_enabled off
[root@localhost sadmin]# systemctl restart nginx
Job for nginx.service failed because the control process exited with error code.
See "systemctl status nginx.service" and "journalctl -xeu nginx.service" for details.
```
Разрешим в SELinux работу nginx на порту TCP 4881 c помощью формирования и установки модуля SELinux
```
root@localhost nginx]# grep nginx /var/log/audit/audit.log | audit2allow -M nginx
******************** ВАЖНО ***********************
Чтобы сделать этот пакет политики активным, выполните:

semodule -i nginx.pp

# применяем модуль semodule -i nginx.pp
semodule -i nginx.pp

# перезапускаем nginx и проверяем
[root@localhost sadmin]# systemctl restart nginx
[root@localhost sadmin]# ss -tlnp | grep nginx
LISTEN 0      511          0.0.0.0:4093      0.0.0.0:*    users:(("nginx",pid=232429,fd=6),("nginx",pid=232428,fd=6))
LISTEN 0      511             [::]:4093         [::]:*    users:(("nginx",pid=232429,fd=7),("nginx",pid=232428,fd=7))
[root@localhost sadmin]#

```

отлключаем vjlekm
```
semodule -r nginx
libsemanage.semanage_direct_remove_key: Removing last nginx module (no other nginx module exists at another priority).
```


## Задание 2 Обеспечить работоспособность приложения при включенном selinux.

- развернуть приложенный стенд https://github.com/mbfx/otus-linux-adm/tree/master/selinux_dns_problems;
- выяснить причину неработоспособности механизма обновления зоны (см. README);
- предложить решение (или решения) для данной проблемы;
- выбрать одно из решений для реализации, предварительно обосновав выбор;
- реализовать выбранное решение и продемонстрировать его работоспособность.


Ставим ansible и клонируем репозиторий
```
root@lp-ubn1:/home/sadmin# pipx install --include-deps ansible
  installed package ansible 13.7.0, installed using Python 3.12.3
  These apps are now globally available
    - ansible
    - ansible-community
    - ansible-config
    - ansible-console
    - ansible-doc
    - ansible-galaxy
    - ansible-inventory
    - ansible-playbook
    - ansible-pull
    - ansible-test
    - ansible-vault
⚠️  Note: '/root/.local/bin' is not on your PATH environment variable. These apps will not be globally accessible until your PATH is updated. Run `pipx
    ensurepath` to automatically add it, or manually modify your PATH in your shell's config file (i.e. ~/.bashrc).
done! ✨ 🌟 ✨
root@lp-ubn1:/home/sadmin# git clone https://github.com/Nickmob/vagrant_selinux_dns_problems.git
Cloning into 'vagrant_selinux_dns_problems'...
remote: Enumerating objects: 32, done.
remote: Counting objects: 100% (32/32), done.
remote: Compressing objects: 100% (21/21), done.
remote: Total 32 (delta 9), reused 29 (delta 9), pack-reused 0 (from 0)
Receiving objects: 100% (32/32), 7.23 KiB | 740.00 KiB/s, done.
Resolving deltas: 100% (9/9), done.
```

В моейм слечае есть два сервера:
- 192.168.50.232 - здесь ansible и он же будет сервером dns
- 192.168.50.231 - это будет клиент


мой inventory.ini + плюс немного пришлось поменять playbook(заменить в одном месте пользователя vagrant на sadmin
```
[all]
ns01 ansible_host=192.168.50.232 ansible_user=sadmin ansible_password=123wer ansible_become_password=123wer
client ansible_host=192.168.50.231 ansible_user=sadmin ansible_password=123wer ansible_become_password=123wer

[ns01]
ns01 ansible_host=192.168.50.232 ansible_user=sadmin ansible_password=123wer ansible_become_password=123wer

[client]
client ansible_host=192.168.50.231 ansible_user=sadmin ansible_password=123wer ansible_become_password=123wer



```

так же пришлось поменять /etc/named.conf на свои ip адреса



Пробуем внести изменения в зону, получаем ошибку
```
[root@localhost ~]# nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.232
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.231
> send
update failed: SERVFAIL
```

на сервере видим nfrjt
```
type=AVC msg=audit(1780333022.936:5993): avc:  denied  { write } for  pid=247502 comm="isc-net-0000" name="dynamic" dev="dm-0" ino=103129092 scontext=system_u:system_r:named_t:s0 tcontext=unconfined_u:object_r:named_conf_t:s0 tclass=dir permissive=0

        Was caused by:
                Missing type enforcement (TE) allow rule.

                You can use audit2allow to generate a loadable module to allow this access.

type=AVC msg=audit(1780662628.365:6325): avc:  denied  { write } for  pid=247502 comm="isc-net-0000" name="dynamic" dev="dm-0" ino=103129092 scontext=system_u:system_r:named_t:s0 tcontext=unconfined_u:object_r:named_conf_t:s0 tclass=dir permissive=0

        Was caused by:
                Missing type enforcement (TE) allow rule.

                You can use audit2allow to generate a loadable module to allow this access.

```

видим что как раз используется named_conf_t

```
[root@localhost ~]# ls -laZ /etc/named
итого 28
drw-rwx---.   3 root named system_u:object_r:named_conf_t:s0      121 мая 31 21:48 .
drwxr-xr-x. 136 root root  system_u:object_r:etc_t:s0            8192 мая 31 21:49 ..
drw-rwx---.   2 root named unconfined_u:object_r:named_conf_t:s0   56 мая 31 21:48 dynamic
-rw-rw----.   1 root named system_u:object_r:named_conf_t:s0      784 мая 31 21:48 named.50.168.192.rev
-rw-rw----.   1 root named system_u:object_r:named_conf_t:s0      610 мая 31 21:48 named.dns.lab
-rw-rw----.   1 root named system_u:object_r:named_conf_t:s0      609 мая 31 21:48 named.dns.lab.view1
-rw-rw----.   1 root named system_u:object_r:named_conf_t:s0      657 мая 31 21:48 named.newdns.lab

```

Изменим тип контекста безопасности для каталога /etc/named: sudo chcon -R -t named_zone_t /etc/named
```
[root@localhost ~]# sudo chcon -R -t named_zone_t /etc/named
[root@localhost ~]#
[root@localhost ~]#
[root@localhost ~]#
[root@localhost ~]# ls -laZ /etc/named
итого 28
drw-rwx---.   3 root named system_u:object_r:named_zone_t:s0      121 мая 31 21:48 .
drwxr-xr-x. 136 root root  system_u:object_r:etc_t:s0            8192 мая 31 21:49 ..
drw-rwx---.   2 root named unconfined_u:object_r:named_zone_t:s0   56 мая 31 21:48 dynamic
-rw-rw----.   1 root named system_u:object_r:named_zone_t:s0      784 мая 31 21:48 named.50.168.192.rev
-rw-rw----.   1 root named system_u:object_r:named_zone_t:s0      610 мая 31 21:48 named.dns.lab
-rw-rw----.   1 root named system_u:object_r:named_zone_t:s0      609 мая 31 21:48 named.dns.lab.view1
-rw-rw----.   1 root named system_u:object_r:named_zone_t:s0      657 мая 31 21:48 named.newdns.lab

```

еще раз пробуем обновить зону
```
[sadmin@localhost ~]$ nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.232
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.231
> send
> quit
```
проверяем обновилась ли зона
```
[sadmin@localhost ~]$ dig @192.168.50.232 www.ddns.lab

; <<>> DiG 9.16.23-RH <<>> @192.168.50.232 www.ddns.lab
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 24503
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: 44d5df7245abc474010000006a22c60395968ba7059bf839 (good)
;; QUESTION SECTION:
;www.ddns.lab.                  IN      A

;; ANSWER SECTION:
www.ddns.lab.           60      IN      A       192.168.50.231

;; Query time: 0 msec
;; SERVER: 192.168.50.232#53(192.168.50.232)
;; WHEN: Fri Jun 05 15:50:11 MSK 2026
;; MSG SIZE  rcvd: 85
```

перезагружаем сервера , изменения сохранились
```
Last login: Fri Jun  5 16:02:16 2026 from 192.168.20.25
[sadmin@localhost ~]$ dig @192.168.50.232 www.ddns.lab

; <<>> DiG 9.16.23-RH <<>> @192.168.50.232 www.ddns.lab
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 2113
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: 7541e64af2ed55f8010000006a22c97f13343b855f058f5b (good)
;; QUESTION SECTION:
;www.ddns.lab.                  IN      A

;; ANSWER SECTION:
www.ddns.lab.           60      IN      A       192.168.50.231

;; Query time: 0 msec
;; SERVER: 192.168.50.232#53(192.168.50.232)
;; WHEN: Fri Jun 05 16:05:03 MSK 2026
;; MSG SIZE  rcvd: 85

```
выполнили перемаркировку , все отлитело на первоначальный контекст
```
[root@localhost sadmin]# restorecon -v -R /etc/named
Relabeled /etc/named from system_u:object_r:named_zone_t:s0 to system_u:object_r:named_conf_t:s0
Relabeled /etc/named/named.dns.lab from system_u:object_r:named_zone_t:s0 to system_u:object_r:named_conf_t:s0
Relabeled /etc/named/named.dns.lab.view1 from system_u:object_r:named_zone_t:s0 to system_u:object_r:named_conf_t:s0
Relabeled /etc/named/dynamic from unconfined_u:object_r:named_zone_t:s0 to unconfined_u:object_r:named_conf_t:s0
Relabeled /etc/named/dynamic/named.ddns.lab from system_u:object_r:named_zone_t:s0 to system_u:object_r:named_conf_t:s0
Relabeled /etc/named/dynamic/named.ddns.lab.view1 from system_u:object_r:named_zone_t:s0 to system_u:object_r:named_conf_t:s0
Relabeled /etc/named/dynamic/named.ddns.lab.jnl from system_u:object_r:named_zone_t:s0 to system_u:object_r:named_conf_t:s0
Relabeled /etc/named/named.newdns.lab from system_u:object_r:named_zone_t:s0 to system_u:object_r:named_conf_t:s0
Relabeled /etc/named/named.50.168.192.rev from system_u:object_r:named_zone_t:s0 to system_u:object_r:named_conf_t:s0
```









  
  
