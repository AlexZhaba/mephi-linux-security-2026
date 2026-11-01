# ДЗ №1. Изучение средств защиты ОС GNU/Linux

Александр Жаворонков, М26-545. Номер студенческого: M265846. Номер в группе: 54.

Поставил Debian 13 ARM64 в UTM, машину назвал mephi-lab-01. Работал под `aleksandr`.

## 1. Создание пользователя

Создал группу students и пользователя `user1`:

```bash
sudo groupadd students
sudo useradd -m -u 1234 -G students user1
sudo usermod -s /bin/bash user1
sudo chage -M 90 user1
```

Проверил через id user1:

```text
uid=1234(user1) gid=1234(user1) groups=1234(user1),1001(students)
```

Запись в passwd:

```text
user1:x:1234:1234::/home/user1:/bin/bash
```

Посмотрел срок пароля через `chage -l user1`:

```text
Last password change                                     : Oct 31, 2026
Password expires                                        : Jan 29, 2027
Maximum number of days between password change           : 90
```

## 2. Мониторинг файлов и процессов

Файлы с set-UID искал по всей системе:

```bash
sudo find / -type f -perm -4000
```

Получил список:

```text
/usr/bin/umount
/usr/bin/chsh
/usr/bin/gpasswd
/usr/bin/mount
/usr/bin/sudo
/usr/bin/su
/usr/bin/passwd
/usr/bin/newgrp
/usr/bin/chfn
/usr/lib/openssh/ssh-keysign
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
```

Сначала поискал процессы с ненулевым RUID и EUID 0:

```bash
ps -eo pid,ruid,euid,comm | awk '$2!=0 && $3==0'
```

Первый поиск ничего не вывел. В отдельном терминале запустил смену пароля, затем повторил тот же поиск. В результате появилась строка:

```text
   1948  1000     0 sudo
```

Столбцы: PID, RUID, EUID и команда. UID 1000 принадлежит aleksandr, EUID у процесса равен 0.

## 3. Выполнение операции через set-UID

Попробовал прочитать /etc/shadow от user1:

```bash
sudo -u user1 cat /etc/shadow
```

Получил отказ:

```text
cat: /etc/shadow: Permission denied
```

Сделал копию `cat2` и включил set-UID:

```bash
sudo install -o root -g root -m 0755 /usr/bin/cat /home/user1/cat2
sudo chmod u+s /home/user1/cat2
```

Посмотрел права через ls -l:

```text
-rwsr-xr-x 1 root root 68088 Oct 31 22:28 /home/user1/cat2
```

Повторил чтение:

```bash
sudo -u user1 /home/user1/cat2 /etc/shadow
```

В выводе появилась строка пользователя:

```text
user1:!:20757:0:90:7:::
```

## 4. Выполнение операции через capabilities

Создал файл от user1 и попробовал сменить владельца:

```bash
sudo -u user1 touch /home/user1/some-file.txt
sudo -u user1 chown root:root /home/user1/some-file.txt
```

Команда вернула ошибку:

```text
chown: changing ownership of '/home/user1/some-file.txt': Operation not permitted
```

Сделал копию chown и назначил capability:

```bash
sudo install -o root -g root -m 0755 /usr/bin/chown /home/user1/chown2
sudo setcap cap_chown=ep /home/user1/chown2
```

Проверил через `getcap /home/user1/chown2`:

```text
/home/user1/chown2 cap_chown=ep
```

Повторил команду через копию:

```bash
sudo -u user1 /home/user1/chown2 root:root /home/user1/some-file.txt
```

Посмотрел владельца файла:

```bash
sudo ls -l /home/user1/some-file.txt
```

Получил:

```text
-rw-rw-r-- 1 root root 0 Oct 31 22:37 /home/user1/some-file.txt
```

## 5. Настройка sudo

Открыл `sudo EDITOR=vim visudo` и добавил правило:

```sudoers
user1 ALL=(root) NOPASSWD: /usr/bin/timedatectl set-time *
```

Проверил синтаксис через sudo visudo -c:

```text
/etc/sudoers: parsed OK
/etc/sudoers.d/README: parsed OK
```

Перед ручной установкой отключил синхронизацию времени:

```bash
sudo timedatectl set-ntp false
```

До изменения `date` показывала:

```text
Sat Oct 31 10:59:33 PM MSK 2026
```

Установил дату 15 марта 2027 года:

```bash
sudo -u user1 sudo timedatectl set-time "2027-03-15 12:34:56"
```

Первый sudo переключает на user1, второй использует его разрешение на установку времени.

После изменения `date` показала:

```text
Mon Mar 15 12:35:54 PM MSK 2027
```

Затем выполнил `timedatectl`:

```text
               Local time: Mon 2027-03-15 12:36:36 MSK
           Universal time: Mon 2027-03-15 09:36:36 UTC
                 RTC time: Mon 2027-03-15 09:36:37
                Time zone: Europe/Moscow (MSK, +0300)
System clock synchronized: no
              NTP service: inactive
          RTC in local TZ: no
```
