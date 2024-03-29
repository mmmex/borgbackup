### Backups

```
Настроить стенд Vagrant с двумя виртуальными машинами: backup_server и client.
Настроить удаленный бекап каталога /etc c сервера client при помощи borgbackup. Резервные копии должны соответствовать следующим критериям:

* директория для резервных копий /var/backup. Это должна быть отдельная точка монтирования. В данном случае для демонстрации размер не принципиален, достаточно будет и 2GB;
* репозиторий для резервных копий должен быть зашифрован ключом или паролем - на ваше усмотрение;
* имя бекапа должно содержать информацию о времени снятия бекапа;
* глубина бекапа должна быть год, хранить можно по последней копии на конец месяца, кроме последних трех. Последние три месяца должны содержать копии на каждый день. Т.е. должна быть правильно настроена политика удаления старых бэкапов;
* резервная копия снимается каждые 5 минут. Такой частый запуск в целях демонстрации;
* написан скрипт для снятия резервных копий. Скрипт запускается из соответствующей Cron джобы, либо systemd timer-а - на ваше усмотрение;
* настроено логирование процесса бекапа. Для упрощения можно весь вывод перенаправлять в logger с соответствующим тегом. Если настроите не в syslog, то обязательна ротация логов.

Запустите стенд на 30 минут.
Убедитесь что резервные копии снимаются.
Остановите бекап, удалите (или переместите) директорию /etc и восстановите ее из бекапа.
Для сдачи домашнего задания ожидаем настроенные стенд, логи процесса бэкапа и описание процесса восстановления.
Формат сдачи ДЗ - vagrant + ansible
```

## Стенд Vagrant с двумя виртуальными машинами: backup_server и client.

* Клонируем репозиторий: `git clone https://github.com/mmmex/borgbackup.git`

* Переходим в директорию `borgbackup` и выполняем запуск: `cd borgbackup; vagrant up`

* Демонстрация запуска:

[![asciicast](https://asciinema.org/a/550237.svg)](https://asciinema.org/a/550237)

* После выполнения стенд полностью готов к демонстрации.

## Настройка резервного копирования каталога /etc с хоста client на удаленный сервер backup_server при помощи borgbackup.

Согласно техническому заданию, Ansible настроит две ВМ: `client` и `backup_server`.

* Демонстрация настроек и выполнения ДЗ:

[![asciicast](https://asciinema.org/a/UoXDXIQm5i94fgeuRqlLdU4qY.svg)](https://asciinema.org/a/UoXDXIQm5i94fgeuRqlLdU4qY)

## Описание процесса восстановления каталога /etc

* Выполним вход на ВМ client: `vagrant ssh client`

* Останавливаем службу таймера резервного копирования: `systemctl stop borg-backup.timer`

* Убедимся что у нас имеется резервная копия:

```bash
[root@client ~]# borg list                                                                                                        
etc-2023-01-08_13:50:14              Sun, 2023-01-08 13:50:16 [0ce09afadd14227fd23b11fe0a0dcae5ae92f6a4d3d52fa56d9d2ce9de7df42f]
```

* Удаляем каталог /etc:

```bash
[root@client ~]# rm -rf /etc                                                                                                      
rm: cannot remove '/etc': Device or resource busy
```

* Проверяем что действительно содержимое каталога отсутствует:

```bash
[root@client ~]# ls -al /etc                                                                                                      
total 0                                                                                                                           
drwxr-xr-x.  2 0 0   6 Jan  8 11:41 .                                                                                             
dr-xr-xr-x. 18 0 0 255 Jan  8 10:03 ..
```

* Пробуем получить перечень копий из репозитория:

```bash
[root@client ~]# borg list                                                                                                        
Connection closed by remote host. Is borg working on the server?
```

* Проблема в том, что теперь ОС не может выполнить соединения с удаленным сервером из-за отсутствия в конфигурации пользователя root, добавляем:

```bash
[root@client ~]# echo "root:x:0:0:root:/root:/bin/bash" > /etc/passwd 
```

* Также в нашем окружении добавлена переменная BORG_REPO, в которой содержится имя хоста и соответственно необходимо его добавить в файл /etc/hosts:

```bash
[root@client ~]# echo "192.168.56.2 server" > /etc/hosts
```

* Теперь можно выполнить запрос на получение доступа к удаленному репозиторию и можно выполнить восстановление из последней копии:

```bash
[root@client ~]# borg list                                                                                                        
etc-2023-01-08_13:50:14              Sun, 2023-01-08 10:50:16 [0ce09afadd14227fd23b11fe0a0dcae5ae92f6a4d3d52fa56d9d2ce9de7df42f]
[root@client ~]# cd /;borg extract ::etc-2023-01-08_13:50:14                                                                      
Warning: File system encoding is "ascii", extracting non-ascii filenames will not be supported.                                   
Hint: You likely need to fix your locale setup. E.g. install locales and use: LANG=en_US.UTF-8
```

* Можем сравнить содержимое архива с локальной папкой:

```bash
[root@client /]# borg export-tar ::etc-2023-01-08_13:50:14 - | tar --compare -f - -C /
```