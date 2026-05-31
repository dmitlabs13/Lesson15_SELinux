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





  
  
