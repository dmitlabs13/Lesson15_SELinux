# Lesson15_SELinux

## Задание1
Запустить nginx на нестандартном порту 3-мя разными способами:
- переключатели setsebool;
- добавление нестандартного порта в имеющийся тип;
- формирование и установка модуля SELinux.


## Выполнение

проверяем что firewall ыключен
```
[root@localhost sadmin]# systemctl status firewalld.service
WARNING: terminal is not fully functional
Press RETURN to continue
● firewalld.service - firewalld - dynamic firewall daemon
     Loaded: loaded (/usr/lib/systemd/system/firewalld.service; enabled; preset: enabled)
     Active: active (running) since Sun 2026-05-03 16:01:56 MSK; 3 weeks 6 days ago
       Docs: man:firewalld(1)
   Main PID: 788 (firewalld)
      Tasks: 2 (limit: 10464)
     Memory: 1.0M (peak: 60.2M)
        CPU: 596ms
     CGroup: /system.slice/firewalld.service
             └─788 /usr/bin/python3 -s /usr/sbin/firewalld --nofork --nopid

```


  
  
