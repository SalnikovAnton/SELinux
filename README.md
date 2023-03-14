# Управление процессами 
### 1) Запустить nginx на нестандартном порту 3-мя разными способами:
* переключатели setsebool;
* добавление нестандартного порта в имеющийся тип;
* формирование и установка модуля SELinux.

### 2) Обеспечить работоспособность приложения при включенном selinux.
* развернуть приложенный стенд https://github.com/mbfx/otus-linux-adm/tree/master/selinux_dns_problems; 
* выяснить причину неработоспособности механизма обновления зоны (см. README);
* предложить решение (или решения) для данной проблемы;
* выбрать одно из решений для реализации, предварительно обосновав выбор;
* реализовать выбранное решение и продемонстрировать его работоспособность


### 1) Запускаем вертуальную машину используя [Vagrant-file](https://github.com/SalnikovAnton/SELinux/blob/main/Vagrantfile "Vagrant-file").  
Результатом выполнения команды vagrant up станет созданная виртуальная машина с установленным nginx, который работает на порту TCP 4881. Порт TCP 4881 уже проброшен до хоста. SELinux включен.  
Во время развёртывания стенда попытка запустить nginx завершится с ошибкой:  

```
selinux: Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
    selinux: ● nginx.service - The nginx HTTP and reverse proxy server
    selinux:    Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
    selinux:    Active: failed (Result: exit-code) since Thu 2023-03-02 15:04:53 UTC; 41ms ago
    selinux:   Process: 2771 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
    selinux:   Process: 2770 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
    selinux: 
    selinux: Mar 02 15:04:53 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
    selinux: Mar 02 15:04:53 selinux nginx[2771]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
    selinux: Mar 02 15:04:53 selinux nginx[2771]: nginx: [emerg] bind() to 0.0.0.0:4881 failed (13: Permission denied)
    selinux: Mar 02 15:04:53 selinux nginx[2771]: nginx: configuration file /etc/nginx/nginx.conf test failed
    selinux: Mar 02 15:04:53 selinux systemd[1]: nginx.service: control process exited, code=exited status=1
    selinux: Mar 02 15:04:53 selinux systemd[1]: Failed to start The nginx HTTP and reverse proxy server.
    selinux: Mar 02 15:04:53 selinux systemd[1]: Unit nginx.service entered failed state.
    selinux: Mar 02 15:04:53 selinux systemd[1]: nginx.service failed.
```
Данная ошибка появляется из-за того, что SELinux блокирует работу nginx на нестандартном порту  
Проверим, что в ОС отключен файервол  
```
[root@selinux ~]# systemctl status firewalld
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; disabled; vendor preset: enabled)
   Active: inactive (dead)
     Docs: man:firewalld(1)
```
Проверяем конфигурацию nginx
```
[root@selinux ~]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```
Далее проверим режим работы SELinux
```
[root@selinux ~]# getenforce
Enforcing
```

#### Разрешим в SELinux работу nginx на порту TCP 4881 c помощью переключателей setsebool;  
Находим в логах (/var/log/audit/audit.log) информацию о блокировании порта
```
type=SYSCALL msg=audit(1677769493.338:797): arch=c000003e syscall=49 success=no exit=-13 a0=6 a1=560a2018a878 a2=10 a3=7ffde1413520 items=0 ppid=1 pid=2771 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm="nginx" exe="/usr/sbin/nginx" subj=system_u:system_r:httpd_t:s0 key=(null)
type=SERVICE_START msg=audit(1677769493.355:798): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=failed'
```
Копируем время, в которое был записан этот лог, и, с помощью утилиты audit2why смотрим
```
[root@selinux ~]# grep 1677769493.338:797 /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1677769493.338:797): avc:  denied  { name_bind } for  pid=2771 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0

        Was caused by:
        The boolean nis_enabled was set incorrectly. 
        Description:
        Allow nis to enabled

        Allow access by executing:
        # setsebool -P nis_enabled 1               <<< сходя из вывода утилиты, мы видим, что нам нужно поменять параметр nis_enabled
```
Включаем параметр и проверяем включение
```
[root@selinux ~]# setsebool -P nis_enabled on
[root@selinux ~]# getsebool -a | grep nis_enabled
nis_enabled --> on
```
После включения параметра nis_enabled необходимо перезапустить nginx 
```
[root@selinux ~]# systemctl restart nginx
```
Вернём запрет работы nginx на порту 4881 обратно. Для этого отключим nis_enabled
```
[root@selinux ~]# setsebool -P nis_enabled off
[root@selinux ~]# getsebool -a | grep nis_enabled
nis_enabled --> off
```
#### Разрешим в SELinux работу nginx на порту TCP 4881 c помощью добавления нестандартного порта в имеющийся тип;  
Поиск имеющегося типа, для http трафика
```
[root@selinux ~]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
```
Добавим порт в тип http_port_t и проверяем
```
[root@selinux ~]# semanage port -a -t http_port_t -p tcp 4881
[root@selinux ~]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      4881, 80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
```
Теперь перезапустим службу nginx и проверим её работу
```
[root@selinux ~]# systemctl restart nginx
[root@selinux ~]# systemctl status nginx

[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Tue 2023-03-14 12:10:15 UTC; 12s ago
  Process: 3888 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 3886 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 3884 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 3890 (nginx)
   CGroup: /system.slice/nginx.service
           ├─3890 nginx: master process /usr/sbin/nginx
           └─3891 nginx: worker process
```
Удаляем порт
```
[root@selinux ~]# semanage port -d -t http_port_t -p tcp 4881
[root@selinux ~]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989

[root@selinux ~]# systemctl restart nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Thu 2023-03-02 17:00:09 UTC; 19s ago
  Process: 22000 ExecStart=/usr/sbin/nginx (code=exited, status=1/FAILURE)
  Process: 21998 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 21997 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)

Mar 02 17:00:07 selinux nginx[22000]: nginx: [emerg] bind() to [::]:4881 failed (98: Address already in use)
Mar 02 17:00:08 selinux nginx[22000]: nginx: [emerg] bind() to 0.0.0.0:4881 failed (98: Address already in use)
Mar 02 17:00:08 selinux nginx[22000]: nginx: [emerg] bind() to [::]:4881 failed (98: Address already in use)
Mar 02 17:00:08 selinux nginx[22000]: nginx: [emerg] bind() to 0.0.0.0:4881 failed (98: Address already in use)
Mar 02 17:00:08 selinux nginx[22000]: nginx: [emerg] bind() to [::]:4881 failed (98: Address already in use)
Mar 02 17:00:09 selinux nginx[22000]: nginx: [emerg] still could not bind()
Mar 02 17:00:09 selinux systemd[1]: nginx.service: control process exited, code=exited status=1
Mar 02 17:00:09 selinux systemd[1]: Failed to start The nginx HTTP and reverse proxy server.
Mar 02 17:00:09 selinux systemd[1]: Unit nginx.service entered failed state.
Mar 02 17:00:09 selinux systemd[1]: nginx.service failed.
```
#### Разрешим в SELinux работу nginx на порту TCP 4881 c помощью формирования и установки модуля SELinux:  
Запускаем nginx
```
[root@selinux ~]# systemctl start nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
```
смотрим логи SELinux, которые относятся к nginx
```
type=SYSCALL msg=audit(1677776406.711:897): arch=c000003e syscall=49 success=no exit=-98 a0=6 a1=55dc0ab15878 a2=10 a3=7ffec628f720 items=0 ppid=1 pid=21998 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm="nginx" exe="/usr/sbin/nginx" subj=system_u:system_r:httpd_t:s0 key=(null)
type=SERVICE_START msg=audit(1677776409.297:898): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=failed'
type=SERVICE_START msg=audit(1677776584.252:906): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=failed'
```
Воспользуемся утилитой audit2allow для того, чтобы на основе логов SELinux сделать модуль, разрешающий работу nginx на нестандартном порту
```
[root@selinux ~]# grep nginx /var/log/audit/audit.log | audit2allow -M nginx
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i nginx.pp
```
Audit2allow сформировал модуль, и сообщил нам команду, с помощью которой можно применить данный модуль и перезапускаем nginx
```
[root@selinux ~]# semodule -i nginx.pp
[root@selinux ~]# systemctl restart nginx
[root@selinux ~]# systemctl status nginx

[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Tue 2023-03-14 12:10:15 UTC; 12s ago
  Process: 3888 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 3886 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 3884 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 3890 (nginx)
   CGroup: /system.slice/nginx.service
           ├─3890 nginx: master process /usr/sbin/nginx
           └─3891 nginx: worker process

```
Просмотр всех установленных модулей: semodule -l  
Для удаления модуля воспользуемся командой
```
[root@selinux ~]# semodule -r nginx
```






