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