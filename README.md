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

1. Запуск nginx при помощи:

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
