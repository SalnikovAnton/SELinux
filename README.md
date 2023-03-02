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



















