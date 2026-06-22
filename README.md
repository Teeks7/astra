# Гайд: Astra Linux SE 1.8 + файловый сервер на Samba

*Квалификационный экзамен «Оператор ЭВМ» — установка ОС без GUI, настройка сетевых папок с разграничением доступа по отделам*

---

## Легенда задания

Одноранговая сеть, два отдела (бухгалтерия и продажи), новый сервер используется как файловый сервер. Служба каталогов не используется.

**Требуется:**
1. Установить Astra Linux SE 1.8 без графического интерфейса
2. Создать точку монтирования `/srv`
3. `/srv/share/buh` — доступ на чтение/запись только для бухгалтерии
4. `/srv/share/marketing` — доступ на чтение/запись только для продаж
5. `/srv/share` — общая папка, доступна обоим отделам на чтение/запись
6. Отделы не должны видеть каталоги друг друга

---

## Схема адресации (придумываем сами, если не выдана готовая)

| Машина | IP |
|---|---|
| Сервер | 192.168.1.1/24 |
| ПК бухгалтерии | 192.168.1.10/24 |
| ПК продаж | 192.168.1.20/24 |

---

```bash
whoami
```

Если вы не root — получить root-сессию:
```bash
sudo -i
```

> Если на установке вы **не отключили** мандатный контроль целостности, при логине появится `Integrity level:` — просто нажмите Enter для уровня по умолчанию, либо введите конкретное число (например `63`) при проблемах с правами на запись системных файлов.

---

## Часть 3. Настройка сети (выполняется на сервере и на каждом клиенте, со своим IP)

```bash
ip a
```
Запомнить имя интерфейса (например `enp0s3`).

```bash
nano /etc/network/interfaces
```

Для сервера дописать в конец файла:
```
auto enp0s3
iface enp0s3 inet static
    address 192.168.1.1
    netmask 255.255.255.0
    gateway 192.168.1.254
    dns-nameservers 8.8.8.8
```

Для клиента бухгалтерии — `address 192.168.1.10`, для клиента продаж — `address 192.168.1.20` (остальное аналогично).

Сохранить (`Ctrl+O`, `Enter`, `Ctrl+X`), применить:
```bash
systemctl restart networking
ip a
```

Проверить интернет и связь между машинами:
```bash
ping -c 3 8.8.8.8
ping -c 3 192.168.1.1   # с клиента на сервер
```

---

## Часть 4. Проверка и настройка источников пакетов (на сервере)

```bash
cat /etc/apt/sources.list
```

Убедиться, что строки с `https://download.astralinux.ru/...` **не закомментированы**, а строка с `cdrom:` — закомментирована (`#` в начале). Если нужно редактировать:

```bash
nano /etc/apt/sources.list
```

Если при сохранении возникает ошибка **«Отказано в доступе»** — это почти всегда из-за включённого мандатного контроля целостности (см. Часть 1). Решения:
```bash
lsattr /etc/apt/sources.list      # проверить атрибут immutable
chattr -i /etc/apt/sources.list   # снять, если стоит
mount | grep " / "                # проверить, не примонтирован ли root в ro
mount -o remount,rw /             # перемонтировать в rw при необходимости
```
Либо перезайти с явным указанием уровня целостности (`Integrity level: 63`).

Обновить пакеты:
```bash
apt update
apt upgrade -y
```

---

## Часть 5. Установка Samba (только на сервере)

```bash
apt install samba -y
systemctl status smbd
```

---

## Часть 6. Точка монтирования /srv (на сервере)

```bash
mkdir -p /srv
ls -ld /srv
```

---

## Часть 7. Структура папок (на сервере)

```bash
mkdir -p /srv/share/buh
mkdir -p /srv/share/marketing
ls -la /srv/share/
```

---

## Часть 8. Группы и пользователи (на сервере)

```bash
groupadd buh
groupadd marketing

useradd -m -G buh buh_user
useradd -m -G marketing sales_user

passwd buh_user
passwd sales_user

groups buh_user
groups sales_user
```

---

## Часть 9. Права на директории (на сервере)

```bash
chown root:buh /srv/share/buh
chmod 770 /srv/share/buh

chown root:marketing /srv/share/marketing
chmod 770 /srv/share/marketing

chown root:root /srv/share
chmod 777 /srv/share

ls -la /srv/share/
```

---

## Часть 10. Настройка smb.conf (на сервере)

```bash
cp /etc/samba/smb.conf /etc/samba/smb.conf.bak
nano /etc/samba/smb.conf
```

В конец файла дописать:
```ini
[buh]
   path = /srv/share/buh
   valid users = @buh
   browseable = no
   read only = no
   create mask = 0770
   directory mask = 0770

[marketing]
   path = /srv/share/marketing
   valid users = @marketing
   browseable = no
   read only = no
   create mask = 0770
   directory mask = 0770

[share]
   path = /srv/share
   valid users = @buh, @marketing
   browseable = yes
   read only = no
   create mask = 0777
   directory mask = 0777
```

Проверить синтаксис:
```bash
testparm
```

---

## Часть 11. Samba-пароли пользователей (на сервере)

```bash
smbpasswd -a buh_user
smbpasswd -a sales_user
```

---

## Часть 12. Запуск служб (на сервере)

```bash
systemctl restart smbd nmbd
systemctl enable smbd nmbd
systemctl status smbd nmbd
```

Если `failed`:
```bash
journalctl -u smbd -n 50 --no-pager
```

---

## Часть 13. Firewall (на сервере, если ufw активен)

```bash
ufw status
ufw allow samba
ufw reload
```

---

## Часть 14. Проверка прямо на сервере (без второй машины)

```bash
smbclient -L localhost -U buh_user
```
Ожидается: видны `buh` и `share`, **не виден** `marketing`.

```bash
smbclient -L localhost -U sales_user
```
Ожидается: видны `marketing` и `share`, **не виден** `buh`.

### Полная проверка через монтирование (имитация сетевого доступа локально)

```bash
mkdir -p /mnt/buh /mnt/share /mnt/marketing

mount -t cifs //localhost/buh /mnt/buh -o username=buh_user
mount -t cifs //localhost/share /mnt/share -o username=buh_user

touch /mnt/buh/test.txt
touch /mnt/share/test_from_buh.txt
ls /mnt/buh/
ls /mnt/share/

# проверка запрета доступа в marketing
mount -t cifs //localhost/marketing /mnt/marketing -o username=buh_user
# ожидается: mount error(13): Permission denied
```

---

## Часть 15. Проверка с реальных клиентских машин (если есть вторая/третья ВМ или ПК)

На каждом клиенте после установки ОС и настройки сети (Часть 1-3):

```bash
apt install cifs-utils smbclient -y
```

**С клиента бухгалтерии:**
```bash
smbclient -L //192.168.1.1 -U buh_user

mkdir -p /mnt/buh /mnt/share
mount -t cifs //192.168.1.1/buh /mnt/buh -o username=buh_user
mount -t cifs //192.168.1.1/share /mnt/share -o username=buh_user

touch /mnt/buh/test_buh.txt
touch /mnt/share/test_from_buh.txt

# проверка запрета
mkdir -p /mnt/marketing
mount -t cifs //192.168.1.1/marketing /mnt/marketing -o username=buh_user
# ожидается: Permission denied
```

**С клиента продаж** — аналогично, но с `sales_user`, монтируя `marketing` и `share`, пробуя (неудачно) зайти в `buh`.

**Финальная проверка общей папки:**
```bash
ls /mnt/share/
```
С компа продаж должен быть виден файл, созданный с компа бухгалтерии, и наоборот.

---

## Чек-лист готовности

- [ ] Astra Linux SE 1.8 установлена без графического интерфейса
- [ ] Сеть настроена, есть связь между машинами и (при необходимости) интернет
- [ ] `/srv` существует
- [ ] `/srv/share/buh`, `/srv/share/marketing`, `/srv/share` созданы
- [ ] Группы `buh`, `marketing` созданы, пользователи добавлены
- [ ] Права POSIX выставлены (770/770/777)
- [ ] `smb.conf` содержит три секции с `valid users` и `browseable = no` для отделов
- [ ] Samba-пароли созданы, службы запущены и в автозагрузке
- [ ] Локальная или сетевая проверка подтверждает: каждый отдел видит свою папку и общую, не видит чужую
- [ ] Общая папка доступна на запись из обоих отделов
