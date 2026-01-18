Инструкция «как мы добавили NVMe в OpenWrt One и сделали из него постоянное хранилище»  
(каждая строка = команда + короткий комментарий на русском)

```bash
# 1. Проверяем, увидел ли ядро диск
dmesg | grep -i nvme

# 2. Смотрим список NVMe-устройств
ls -l /dev/nvme*

# 3. Обновляем список пакетов
opkg update

# 4. Ставим нужные утилиты (разметка, ФС, автомонтирование)
opkg install fdisk gdisk e2fsprogs block-mount

# 5. Создаём один primary-раздел на весь диск (авто-ответы через echo)
echo -e "o\nn\np\n1\n\n\nw" | fdisk /dev/nvme0n1

# 6. Форматируем раздел в ext4, метка nvme_data
mkfs.ext4 -L nvme_data /dev/nvme0n1p1

# 7. Создаём точку монтирования
mkdir -p /mnt/nvme

# 8. Монтируем вручную (проверка)
mount /dev/nvme0n1p1 /mnt/nvme

# 9. Смотрим, сколько места доступно
df -h /mnt/nvme

# 10. Генерируем шаблон fstab
block detect > /etc/config/fstab

# 11-17. Настраиваем автоматическое монтирование через UCI
uci set fstab.@mount[0].device='/dev/nvme0n1p1'
uci set fstab.@mount[0].target='/mnt/nvme'
uci set fstab.@mount[0].fstype='ext4'
uci set fstab.@mount[0].options='rw,noatime'
uci set fstab.@mount[0].enabled='1'
uci set fstab.@mount[0].enabled_fsck='0'
uci commit fstab

# 18. Включаем сервис fstab и перезапускаем
/etc/init.d/fstab enable
/etc/init.d/fstab restart

# 19. Проверяем итоговую таблицу
df -h
```

**Для чего всё это:**  

теперь 233 ГБ свободного места всегда доступны в `/mnt/nvme` – кладём туда торренты, бэкапы, Docker-контейнеры, базы AdGuardHome, Samba/NFS-шары и т.д., не трогая встроенную NAND.

--------------------------------
<br/>



## **Анализ блочных устройств:**

Текущие данные
```bash
root@OpenWrt:~# cat  /etc/config/fstab

config global
        option anon_swap '0'
        option anon_mount '0'
        option auto_swap '1'
        option auto_mount '1'
        option delay_root '5'
        option check_fs '0'

config mount
        option target '/mnt/nvme'
        option uuid '1e69df50-8428-4048-a410-6cd33e821777'
        option enabled '1'
        option device '/dev/nvme0n1p1'
        option fstype 'ext4'
        option options 'rw,noatime'
        option enabled_fsck '0'

config mount
        option target '/rom'
        option uuid '54bda554-68d27cb6-759396e4-8ac8f5c0'
        option enabled '0'
```

```bash
root@OpenWrt:~# lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
mtdblock0    31:0    0   256K  0 disk
mtdblock1    31:1    0   768K  1 disk
mtdblock2    31:2    0   512K  0 disk
mtdblock3    31:3    0  12.5M  0 disk
mtdblock4    31:4    0     1M  1 disk
mtdblock5    31:5    0   255M  0 disk
ubiblock0_4 254:0    0  10.3M  0 disk
fit0        259:0    0   4.8M  1 disk /rom
nvme0n1     259:1    0 238.5G  0 disk
└─nvme0n1p1 259:2    0 238.5G  0 part /mnt/nvme
```


### **1. MTD устройства (Memory Technology Device) - флеш-память роутера:**
```c
mtdblock0    31:0    0   256K  0 disk  # Вероятно, bootloader или раздел с DTB
mtdblock1    31:1    0   768K  1 disk  # Скорее всего, загрузчик с резервной копией
mtdblock2    31:2    0   512K  0 disk  # Раздел с конфигурацией загрузчика
mtdblock3    31:3    0  12.5M  0 disk  # Ядро и initramfs (fit0 монтируется отсюда)
mtdblock4    31:4    0     1M  1 disk  # Резервный раздел
mtdblock5    31:5    0   255M  0 disk  # Основная файловая система UBI
```

### **2. UBI (Unsorted Block Images) - основной раздел:**
```
ubiblock0_4 254:0    0  10.3M  0 disk  # Сжатый read-only корневой раздел
fit0        259:0    0   4.8M  1 disk /rom  # Фактически монтированная ОС (read-only)
```

### **3. NVMe диск:**
```
nvme0n1     259:1    0 238.5G  0 disk         # Весь физический диск NVMe
└─nvme0n1p1 259:2    0 238.5G  0 part /mnt/nvme  # Раздел, смонтированный как /mnt/nvme
```

## **Где стоит ОС и как устроена файловая система:**

### **Текущая структура монтирования:**
```
/dev/root                 5.0M      5.0M         0 100% /rom
/dev/ubi0_5             200.9M     24.4M    171.8M  12% /overlay
overlayfs:/overlay      200.9M     24.4M    171.8M  12% /
```

**Объяснение:**
1. **`/rom`** - Read-Only Memory (5MB)
   - Монтируется с `fit0` (из mtdblock3)
   - Содержит базовую систему OpenWRT
   - Нельзя изменять (только для чтения)

2. **`/overlay`** - Overlay файловая система (200.9MB)
   - Монтируется с `ubi0_5` (mtdblock5)
   - Это **ВАША основная ОС** с возможностью записи
   - Все изменения, установка пакетов, конфиги хранятся здесь

3. **`/`** (корневая файловая система)
   - Это overlayfs, которая объединяет `/rom` (read-only) + `/overlay` (read-write)
   - При чтении: сначала проверяется `/overlay`, если файла нет - берется из `/rom`
   - При записи: все идет в `/overlay`

## **Что можно сделать с NVMe для улучшения системы:**

### **Вариант 1: Перенос overlay на NVMe** (рекомендуется):
```bash
# 1. Остановить сервисы, использующие overlay
/etc/init.d/logd stop
/etc/init.d/uhttpd stop

# 2. Скопировать overlay на NVMe
mkdir -p /mnt/nvme/overlay
cp -a /overlay/* /mnt/nvme/overlay/

# 3. Изменить fstab
uci set fstab.overlay="mount"
uci set fstab.overlay.device="/dev/nvme0n1p1"
uci set fstab.overlay.target="/overlay"
uci set fstab.overlay.fstype="ext4"
uci set fstab.overlay.options="rw,noatime"
uci set fstab.overlay.enabled="1"
uci commit fstab

# 4. Или вручную отредактировать /etc/config/fstab:
# config mount
#     option target '/overlay'
#     option device '/dev/nvme0n1p1'
#     option fstype 'ext4'
#     option options 'rw,noatime'
#     option enabled '1'
```

### **Вариант 2: Расширение overlay через симлинки:**
```bash
# Перенести самые объемные директории
mkdir -p /mnt/nvme/overlay_files
mv /overlay/usr/lib /mnt/nvme/overlay_files/
ln -s /mnt/nvme/overlay_files/lib /overlay/usr/lib
```

### **Вариант 3: Использовать NVMe для конкретных служб:**
```bash
# Docker (если будете использовать)
opkg install docker
uci set docker.overlay.path='/mnt/nvme/docker'
uci commit docker

# Torrent клиент
opkg install transmission-daemon
uci set transmission.@transmission[0].download_dir='/mnt/nvme/downloads'
uci commit transmission
```

## **Рекомендации:**
1. **Сейчас ОС работает с флеш-памяти (mtdblock5)**, что ограничивает:
   - Место для пакетов (всего 200MB)
   - Скорость записи
   - Долговечность флеш-памяти

2. **NVMe можно использовать для:**
   - Увеличения overlay (Вариант 1 - лучший)
   - Хранения логов
   - Бекапов
   - Кэширования DNS/web
   - Медиасервера

3. **Проверить текущее использование overlay:**
```bash
du -sh /overlay/* | sort -hr
```

--------------------------------
<br/>








