## OTUS Administrator Linux. Professional ДЗ №12: Практика с SELinux

**Задание**

1. Запустить nginx на нестандартном порту 3-мя разными способами:

- переключатели setsebool;
- добавление нестандартного порта в имеющийся тип;
- формирование и установка модуля SELinux.
  К сдаче:
  README с описанием каждого решения (скриншоты и демонстрация приветствуются).

2. Обеспечить работоспособность приложения при включенном selinux:

   1. развернуть приложенный стенд https://github.com/mbfx/otus-linux-adm/tree/master/selinux_dns_problems;
   2. выяснить причину неработоспособности механизма обновления зоны (см. README);
   3. предложить решение (или решения) для данной проблемы;
   4. выбрать одно из решений для реализации, предварительно обосновав выбор;
   5. реализовать выбранное решение и продемонстрировать его работоспособность.

**_Решение_**

1.  Запуск nginx при помощи:

    1. seboolean:

       1. Посмотрим, почему не запустился nginx:

          ```
          journalctl -xeu nginx

          -- Unit nginx.service has begun starting up.
          selinux nginx[2531]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
          selinux nginx[2531]: nginx: [emerg] bind() to [::]:4881 failed (13: Permission denied)
          selinux nginx[2531]: nginx: configuration file /etc/nginx/nginx.conf test failed
          selinux systemd[1]: nginx.service: control process exited, code=exited status=1
          ```

          Ошибка указывает на невозможность прослушивания порта 4881, и отказе в доступе.

       2. Найдём причину, по которой был заблокирован порт в nginx:

          ```
          ausearch -ts recent|grep 4881|audit2why

          type=AVC msg=audit(1677520944.570:702): avc:  denied  { name_bind } for  pid=2531 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0

              Was caused by:
              The boolean nis_enabled was set incorrectly.
              Description:
              Allow system to run with NIS

              Allow access by executing:
              # setsebool -P nis_enabled 1
          ```

       3. Изменим состояние nis_enabled, перезапустим nginx.
          ```
          setsebool -P nis_enabled 1 && systemctl restart nginx
          ```
       4. Проверим состояние при помощи curl на хост-машине:

          ```
          curl -Is http://localhost:4881|head -n 1
          HTTP/1.1 200 OK
          ```

          Возвращается код 200, поскольку порт 4881 проброшен с хост-машины в ВМ - отвечает nginx изнутри.

       5. Вернём seboolean на значение по умолчанию, что вновь заблокирует работу nginx на нестандартном порту:
          ```
          setsebool -P nis_enabled 0
          ```

    2. Нестандартный порт в имеющийся тип:

       1. Посмотрим, какие порты разрешены для http-трафика:
          ```
          [root@selinux ~]# semanage port -l | grep http_port
          http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
          ```
       2. Просто переназначив порт в nginx.conf на один из стандартных - можно вернуть nginx в работе. Однако, по заданию требуется запуск на порту 4881.
       3. Добавим в стандартный тип наш порт:
          ```
          [root@selinux ~]# semanage port -a -t http_port_t -p tcp 4881
          [root@selinux ~]# semanage port -l | grep http_port
          http_port_t                    tcp      4881, 80, 81, 443, 488, 8008, 8009, 8443, 9000
          ```
       4. Перезапустим nginx, проверим изнутри ВМ:
          ```
          [root@selinux ~]# systemctl restart nginx
          [root@selinux ~]# curl -Is http://localhost:4881|head -n 1
          HTTP/1.1 200 OK
          ```
       5. Вернём назад стандартные настройки типа http_port_t, и перезапустим nginx:
          ```
          semanage port -d -t http_port_t -p tcp 4881
          systemctl restart nginx
          ```

    3. Специальный модуль

       1. Найдём недавние ошибки selinux, отберём относящиеся к nginx, сформируем модуль при помощи утилиты audit2allow:

          ```
          ausearch -ts recent|grep nginx|audit2allow -M nginx
          ******************** IMPORTANT ***********************
          To make this policy package active, execute:

          semodule -i nginx.pp
          ```

       2. Установим модуль, перезапустим nginx и проверим результат локально в ВМ:

          ```
          [root@selinux ~]# semodule -i nginx.pp
          [root@selinux ~]# systemctl restart nginx
          [root@selinux ~]# curl -Is http://localhost:4881|head -n 1
          HTTP/1.1 200 OK
          ```

2.  Диагностика приложения.

    0. Исправленный Vagrantfile для работоспособного стенда расположен в каталоге [по ссылке](./selinux_dns_problems)
    1. После запуска стенда проверим работу с dns на клиенте:

       ```
       nsupdate -k /etc/named.zonetransfer.key
       server 192.168.50.10
       zone ddns.lab
       update add www.ddns.lab. 60 A 192.168.50.15
       send
       ```

    2. Получаем ошибку SERVFAIL, которая говорит о том, что добавить новую A-запись не удалось.

    3. Проверим ошибки SELinux:

       ```
       sudo ausearch -ts recent|audit2why
       ```

    4. Вывод пустой, значит ошибка возникает не на клиенте.
    5. Проверим ошибки SELinux на сервере:

       ```
       [root@ns01 ~]# ausearch -ts recent|audit2why
       type=AVC msg=audit(1677696792.796:1894): avc:  denied  { create } for  pid=5108 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0

          Was caused by:
                Missing type enforcement (TE) allow rule.

                You can use audit2allow to generate a loadable module to allow this access.
       ```

    6. Видим, что не совпадают контексты scontext и tcontext. У процесса 'isc-worker0000' (исполнимый файл /usr/sbin/named) тип контекста named_t, у объекта, к которому он попытался обратиться - etc_t.

    7. Изучим SELinux-контекст у каталога /etc/named, в котором размещены файлы зоны:

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

    8. Видим, что SELinux-контекст действительно etc_t, что не даёт named сохранять/изменять файлы в данном каталоге.
    9. Посмотрим, к каким каталогам у контекста named_t разрешён доступ:

       ```
       [root@ns01 ~]# semanage fcontext -l | grep named_
       /etc/rndc.*                                        regular file       system_u:object_r:named_conf_t:s0
       /var/named(/.*)?                                   all files          system_u:object_r:named_zone_t:s0
       ```

    10. Из данной ситуации есть следующие варианты выхода:

        - изменить SELinux контекст у каталога /etc/named на named_zone_t и файлов в нём;
        - переместить файлы из /etc/named/ в /var/named/, с внесением соответствующих изменений в named.conf;
        - создать модуль, позволяющий процессу с контекстом named_t создавать файлы в каталогах с контекстом etc_t.

        Предположим, что файлы размещены в /etc/named намеренно, и изменять это местоположение нельзя (например, к этим файлам обращаются другие процессы). Поэтому второй вариант не подходит.

        Третий вариант использовать нежелательно, поскольку etc_t является базовым типом, что приведёт к тому, что named получит излишне широкие полномочия на другие каталоги с таким же контекстом.

    11. Изменим контекст у каталога /etc/named:

        ```
        [root@ns01 ~]# chcon -R -t named_zone_t /etc/named
        [root@ns01 ~]# ls -laZ /etc/named
        drw-rwx---. root named system_u:object_r:named_zone_t:s0 .
        drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
        drw-rwx---. root named unconfined_u:object_r:named_zone_t:s0 dynamic
        -rw-rw----. root named system_u:object_r:named_zone_t:s0 named.50.168.192.rev
        -rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab
        -rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab.view1
        -rw-rw----. root named system_u:object_r:named_zone_t:s0 named.newdns.lab
        ```

    12. После этого - изменения на клиенте успешно проходят:

        ```
        [vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
        > server 192.168.50.10
        > zone ddns.lab
        > update add www.ddns.lab. 60 A 192.168.50.15
        > send
        >
        > quit
        [vagrant@client ~]$ dig www.ddns.lab
        ; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.13 <<>> www.ddns.lab
        ;; global options: +cmd
        ;; Got answer:
        ;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 44073
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
        ;; Query time: 2 msec
        ;; SERVER: 192.168.50.10#53(192.168.50.10)
        ;; WHEN: Wed Mar 01 19:56:08 UTC 2023
        ;; MSG SIZE  rcvd: 96
        ```

    13. После перезагрузки стенда - dns продолжает работать:

        ```
        [vagrant@client ~]$ dig @192.168.50.10 www.ddns.lab
        ; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.13 <<>> @192.168.50.10 www.ddns.lab
        ; (1 server found)
        ;; global options: +cmd
        ;; Got answer:
        ;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 23669
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
        ;; Query time: 11 msec
        ;; SERVER: 192.168.50.10#53(192.168.50.10)
        ;; WHEN: Wed Mar 01 20:01:46 UTC 2023
        ;; MSG SIZE  rcvd: 96
        ```

    14. Восстановить настройки контекста по умолчанию (при этом вновь выведя из строя named) можно командой:
        ```
        restorecon -v -R /etc/named
        ```
