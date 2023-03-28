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

### 2) Обеспечение работоспособности приложения при включенном SELinux.  
Выполним клонирование репозитория  
```
anton@anton-VirtualBox:~/Ansible$ git clone https://github.com/mbfx/otus-linux-adm.git
Клонирование в «otus-linux-adm»...
remote: Enumerating objects: 558, done.
remote: Counting objects: 100% (456/456), done.
remote: Compressing objects: 100% (303/303), done.
remote: Total 558 (delta 125), reused 396 (delta 74), pack-reused 102
Получение объектов: 100% (558/558), 1.38 МиБ | 2.26 МиБ/с, готово.
Определение изменений: 100% (140/140), готово.
```
Развернём 2 ВМ с помощью vagrant
```
anton@anton-VirtualBox:~/Ansible/otus-linux-adm/selinux_dns_problems$ vagrant up
Bringing machine 'ns01' up with 'virtualbox' provider...
Bringing machine 'client' up with 'virtualbox' provider...
==> ns01: Importing base box 'centos/7'...
==> ns01: Matching MAC address for NAT networking...
==> ns01: Checking if box 'centos/7' version '2004.01' is up to date...
***
```
После того, как стенд развернется, проверим ВМ
```
anton@anton-VirtualBox:~/Ansible/otus-linux-adm/selinux_dns_problems$ vagrant status
Current machine states:

ns01                      running (virtualbox)
client                    running (virtualbox)

This environment represents multiple VMs. The VMs are all listed
above with their current state. For more information about a specific
VM, run `vagrant status NAME`.
```
Подключимся к клиенту
```
anton@anton-VirtualBox:~/Ansible/otus-linux-adm/selinux_dns_problems$ vagrant ssh client
Last login: Tue Mar 28 15:56:31 2023 from 10.0.2.2
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
Попробуем внести изменения в зону
```
[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
update failed: SERVFAIL
```
Не удалось. Смотрим логи SELinux
```
[vagrant@client ~]$ sudo -i
[root@client ~]# cat /var/log/audit/audit.log | audit2why
[root@client ~]# 
```
ошибок нет, тогда смотрим на стороне сервера
```
anton@anton-VirtualBox:~/Ansible/otus-linux-adm/selinux_dns_problems$ vagrant ssh ns01
Last login: Tue Mar 28 15:50:00 2023 from 10.0.2.2
[vagrant@ns01 ~]$ sudo -i
[root@ns01 ~]# cat /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1680018601.749:1867): avc:  denied  { write } for  pid=5071 comm="isc-worker0000" name="named" dev="sda1" ino=67542992 scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:named_zone_t:s0 tclass=dir permissive=0

        Was caused by:
        The boolean named_write_master_zones was set incorrectly. 
        Description:
        Allow named to write master zones

        Allow access by executing:
        # setsebool -P named_write_master_zones 1
type=AVC msg=audit(1680021420.564:1901): avc:  denied  { create } for  pid=5071 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0

        Was caused by:
                Missing type enforcement (TE) allow rule.

                You can use audit2allow to generate a loadable module to allow this access.
```
Видим, что ошибка в контексте безопасности. Вместо типа named_t используется тип etc_t   
Проверим данную проблему в каталоге /etc/named
```
[root@ns01 ~]# ls -laZ /etc/named
drw-rwx---. root named system_u:object_r:etc_t:s0       .
drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
drw-rwx---. root named unconfined_u:object_r:etc_t:s0   dynamic
-rw-rw----. root named system_u:object_r:etc_t:s0       named.50.168.192.rev
-rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab
-rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab.view1
-rw-rw----. root named system_u:object_r:etc_t:s0       named.newdns.lab
```
Видим, что контекст безопасности неправильный. Проблема заключается в том, что конфигурационные файлы лежат в другом каталоге.   
Смотрим в каком каталоги должны лежать, файлы, чтобы на них распространялись правильные политики SELinux
```
[root@ns01 ~]# sudo semanage fcontext -l | grep named
/etc/rndc.*                                        regular file       system_u:object_r:named_conf_t:s0 
/var/named(/.*)?                                   all files          system_u:object_r:named_zone_t:s0 
/etc/unbound(/.*)?                                 all files          system_u:object_r:named_conf_t:s0 
***
```
Изменим тип контекста безопасности для каталога /etc/named
```
[root@ns01 ~]# sudo chcon -R -t named_zone_t /etc/named
[root@ns01 ~]# ls -laZ /etc/named
drw-rwx---. root named system_u:object_r:named_zone_t:s0 .
drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
drw-rwx---. root named unconfined_u:object_r:named_zone_t:s0 dynamic
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.50.168.192.rev
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab.view1
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.newdns.lab
```
Попробуем снова внести изменения с клиента
```
[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
update failed: SERVFAIL
> quit
```
Смотрим лог на сервере
```
[root@ns01 ~]# audit2why < /var/log/audit/audit.log
type=AVC msg=audit(1680029535.364:1928): avc:  denied  { write } for  pid=5078 comm="isc-worker0000" name="dynamic" dev="sda1" ino=5325747 scontext=system_u:system_r:named_t:s0 tcontext=unconfined_u:object_r:named_zone_t:s0 tclass=dir permissive=0

        Was caused by:
        The boolean named_write_master_zones was set incorrectly. 
        Description:
        Allow named to write master zones

        Allow access by executing:
        # setsebool -P named_write_master_zones 1
[root@ns01 ~]# audit2allow --debug < /var/log/audit/audit.log


#============= named_t ==============

#!!!! This avc can be allowed using the boolean 'named_write_master_zones'
allow named_t named_zone_t:dir write;
```
Выполняем рекомендованную команду audit2allow
```
[root@ns01 ~]# setsebool -P named_write_master_zones=1
```
Попробуем снова внести изменения с клиента
```
[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
> quit
```
Все удачно, проверяем
```
[vagrant@client ~]$ dig www.ddns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.13 <<>> www.ddns.lab
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 62891
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.ddns.lab.                  IN      A

;; ANSWER SECTION:
www.ddns.lab.           60      IN      A       192.168.50.15

;; AUTHORITY SECTION:
ddns.lab.               3600    IN      NS      ns01.dns.lab.

;; ADDITIONAL SECTION:
ns01.dns.lab.           3600    IN      A       192.168.50.10

;; Query time: 49 msec
;; SERVER: 192.168.50.10#53(192.168.50.10)
;; WHEN: Tue Mar 28 19:33:24 UTC 2023
;; MSG SIZE  rcvd: 96
```
изменения применились. Перезагружаемся



```

```


