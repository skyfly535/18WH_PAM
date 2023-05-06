# Пользователи и группы. Авторизация и аутентификация.

Цель:

Управлять правами с помощью sudo, umask. sgid, suid и более сложными инструментами как PAM и ACL, PolicyKit..

1) Запретить всем пользователям, кроме группы admin логин в выходные (суббота и воскресенье), без учета праздников.
2) Дать конкретному пользователю права работать с докером и возможность рестартить докер сервис.

## Развертывание стенда для демострации работы подключаемых модулей аутентификации (PAM).

Стэнд состоит из хостовой машины под управлением ОС Ubuntu 20.04 на которой развернуты Vagrant виртуальной машины `pam` с учтановленной на него `bento/centos-8.4` (`centos/stream8` из Vagrantfile методички недоступен).

Все дейсвия и настройки (которые описаны в методичке) необходимые для работы стенда были реализованы функциями и методами `Vagrant`.

Все коментарии по каждому блоку указаны в тексте `Vagrantfile`.


Выполняем установку `pam`

```
vagrant up
```

## Результат работы

После развертывания стенда, проверяе работоспособность. 

Так как проверка работоспособности проводилась в пятницу, то для проверки корректности работы стенда я дополнил услови проверки дня недели в скрипте `login.sh` ущё одним условием `[ $(date +%a) = "Fri" ]`.

```
[otusadm@pam vagrant]$ date
Fri May  5 13:27:34 UTC 2023
```

Пробуем сначала подключится по ssh под пользователем `otus`, попытка завершается ошибкой `exit code 1`. Подключение под пользователем `otusadm` происходит без проблем.

```
root@root-ubuntu:/home/roman/project# ssh otus@192.168.56.10
otus@192.168.56.10's password: 
/usr/local/bin/login.sh failed: exit code 1
Connection closed by 192.168.56.10 port 22
root@root-ubuntu:/home/roman/project# ssh otusadm@192.168.56.10
otusadm@192.168.56.10's password: 

This system is built by the Bento project by Chef Software
More information can be found at https://github.com/chef/bento
[otusadm@pam ~]$ client_loop: send disconnect: Broken pipe
```
Закоментируем в файле `sshd` строку с модулем pam_exec  `account    required     pam_exec.so  /usr/local/bin/login.sh`. После этого любой из созданных нами пользователей может подключится по ssh.

```
root@root-ubuntu:/home/roman/project# ssh otus@192.168.56.10
otus@192.168.56.10's password: 

This system is built by the Bento project by Chef Software
More information can be found at https://github.com/chef/bento
[otus@pam ~]$ exit
logout
Connection to 192.168.56.10 closed.
root@root-ubuntu:/home/roman/project# ssh otusadm@192.168.56.10
otusadm@192.168.56.10's password: 

This system is built by the Bento project by Chef Software
More information can be found at https://github.com/chef/bento
[otusadm@pam ~]$ exit
logout
Connection to 192.168.56.10 closed.
```
## Наделение пользователя (otus) права работать с докером и возможность рестартить докер сервис

Устанавливаем docker и пробуем запустить сервис из под нашего текущего пользователя

```
[otus@pam ~]$ systemctl status docker
● docker.service - Docker Application Container Engine
   Loaded: loaded (/usr/lib/systemd/system/docker.service; disabled; vendor preset: disabled)
   Active: inactive (dead)
     Docs: https://docs.docker.com
[otus@pam ~]$ systemctl start docker
==== AUTHENTICATING FOR org.freedesktop.systemd1.manage-units ====
Authentication is required to start 'docker.service'.
Authenticating as: root
Password: 
[otus@pam ~]$ 
```
Смотрим группы в которых состоит наш пользователь.
```
[otus@pam ~]$ groups
otus
```
Лучше, чем официальная документация для решения какой-либо проблемы нет.

Демон Docker привязывается к сокету Unix, а не к порту TCP. По умолчанию сокетом Unix владеет `root` пользователь, и другие пользователи могут получить к нему доступ только с помощью `sudo`. Демон Docker всегда запускается от имени `root` пользователя.

Если вы не хотите предварять docker команду `sudo`, создайте группу Unix под названием `docker` и добавьте в нее пользователей. Когда запускается демон Docker, он создает сокет Unix, доступный членам `docker` группы. В некоторых дистрибутивах Linux система автоматически создает эту группу при установке Docker Engine с помощью менеджера пакетов. В этом случае вам не нужно вручную создавать группу.

`docker` Группа предоставляет пользователю привилегии корневого уровня. 

Следуем по инструкции:
https://docs.docker.com/engine/install/linux-postinstall/

Создавать группу `docker` не пришлось, она была создана автоматически при установке софта Docker.

При добавлении пользователя `otus` в группу `docker`

```
sudo usermod -aG docker $USER
```

возникла следующая ошибка `user is not in the sudoers file`

Команда sudo позволяет обычным пользователям выполнять программы от имени суперпользователя со всеми его правами. Использовать команду sudo могут далеко не все пользователи, а только те, которые указаны в файле /etc/sudoers. Это сообщение об ошибке говорит буквально следующее - вашего пользователя нет в файле sudoers, а значит доступ ему к утилите будет запрещен, а об этом инциденте будет сообщено администратору.

https://losst.pro/oshibka-user-is-not-in-the-sudoers-file-v-ubuntu

В большинстве случаев в файле sudoers настроено так, что утилиту могут использовать все пользователи из группы wheel или sudo. Поэтому достаточно добавить нашего пользователя в эту группу. Для этого используйте команду usermod.

```
usermod -a -G wheel otus
```
Перегружаем машину, проверяем ,в как группах состоит наш пользователь

```
[otus@pam ~]$ groups
otus wheel docker
```
Далее пробуем поуправлять сервисом Docker
```
[otus@pam ~]$ systemctl start docker
==== AUTHENTICATING FOR org.freedesktop.systemd1.manage-units ====
Authentication is required to start 'docker.service'.
Authenticating as: otus
Password: 
==== AUTHENTICATION COMPLETE ====
[otus@pam ~]$ systemctl status docker
● docker.service - Docker Application Container Engine
   Loaded: loaded (/usr/lib/systemd/system/docker.service; disabled; vendor preset: disabled)
   Active: active (running) since Sat 2023-05-06 09:52:49 UTC; 3s ago
     Docs: https://docs.docker.com
 Main PID: 4688 (dockerd)
    Tasks: 8
   Memory: 121.3M
   CGroup: /system.slice/docker.service
           └─4688 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
```

```
[otusadm@pam ~]$ systemctl enable docker.service
==== AUTHENTICATING FOR org.freedesktop.systemd1.manage-unit-files ====
Authentication is required to manage system service or unit files.
Authenticating as: otus
Password: 
==== AUTHENTICATION COMPLETE ====
Created symlink /etc/systemd/system/multi-user.target.wants/docker.service → /usr/lib/systemd/system/docker.service.
==== AUTHENTICATING FOR org.freedesktop.systemd1.reload-daemon ====
Authentication is required to reload the systemd state.
Authenticating as: otus
Password: 
==== AUTHENTICATION COMPLETE ====
[otusadm@pam ~]$ systemctl status docker
● docker.service - Docker Application Container Engine
   Loaded: loaded (/usr/lib/systemd/system/docker.service; enabled; vendor preset: disabled)
   Active: active (running) since Sat 2023-05-06 09:19:19 UTC; 1min 56s ago
     Docs: https://docs.docker.com
 Main PID: 1527 (dockerd)
    Tasks: 8
   Memory: 123.8M
   CGroup: /system.slice/docker.service
           └─1527 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock

```

P.S.: Есть подозрение, что включение пользователя в группу `wheel` и так позволило бы решить эту проблему без дополнительных телодвижений.
