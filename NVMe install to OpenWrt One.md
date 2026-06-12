## Добавление NVMe диска в OpenWrt 25.12.4 (с учётом APK)

**Важно:** В версии 25.12.4 используется пакетный менеджер `apk` вместо `opkg`. Все команды ниже адаптированы под новую систему.

---

### Предварительная проверка

```bash
# 1. Проверяем, увидел ли ядро NVMe контроллер
dmesg | grep -i nvme

# 2. Проверяем, виден ли диск на PCIe шине
cat /proc/bus/pci/devices | grep -i 1e4b

# 3. Смотрим, какие блочные устройства уже есть
lsblk
```

---

### Установка драйвера и утилит

```bash
# 4. Обновляем списки пакетов (аналог opkg update)
apk update

# 5. Устанавливаем драйвер NVMe (КРИТИЧЕСКИ ВАЖНО!)
apk add kmod-nvme

# 6. Устанавливаем утилиты для работы с дисками и файловой системой
apk add fdisk gdisk e2fsprogs block-mount

# 7. Загружаем драйвер вручную (если не загрузился автоматически)
modprobe nvme

# 8. Проверяем, появился ли диск
lsblk
# Должны увидеть: nvme0n1 и nvme0n1p1 (если раздел уже был)
```

---

### Создание раздела (если диск новый)

```bash
# 9. Создаём один primary-раздел на весь диск
fdisk /dev/nvme0n1
# В интерактивном режиме:
# n (новый раздел)
# p (primary)
# 1 (номер раздела)
# Enter (начало по умолчанию)
# Enter (конец по умолчанию)
# w (записать изменения)

# Или автоматически (без интерактива):
echo -e "o\nn\np\n1\n\n\nw" | fdisk /dev/nvme0n1
```

---

### Форматирование раздела

```bash
# 10. Форматируем раздел в ext4 с меткой
mkfs.ext4 -L nvme_data /dev/nvme0n1p1

# 11. Проверяем созданную файловую систему
blkid /dev/nvme0n1p1
```

---

### Ручное монтирование

```bash
# 12. Создаём точку монтирования
mkdir -p /mnt/nvme

# 13. Монтируем вручную для проверки
mount /dev/nvme0n1p1 /mnt/nvme

# 14. Проверяем, сколько места доступно
df -h /mnt/nvme

# 15. Проверяем права доступа
ls -la /mnt/nvme/
```

---

### Настройка автоматического монтирования

```bash
# 16. Генерируем шаблон fstab
block detect > /etc/config/fstab.new

# 17. Просматриваем сгенерированный шаблон
cat /etc/config/fstab.new

# 18. Добавляем в основной конфиг (если нужно)
cat /etc/config/fstab.new >> /etc/config/fstab

# 19. Или настраиваем через UCI (рекомендуется)
uci set fstab.@mount[0]=mount
uci set fstab.@mount[0].device='/dev/nvme0n1p1'
uci set fstab.@mount[0].target='/mnt/nvme'
uci set fstab.@mount[0].fstype='ext4'
uci set fstab.@mount[0].options='rw,noatime'
uci set fstab.@mount[0].enabled='1'
uci set fstab.@mount[0].enabled_fsck='0'
uci commit fstab

# 20. Включаем и запускаем сервис монтирования
/etc/init.d/fstab enable
/etc/init.d/fstab restart

# 21. Проверяем, что диск примонтировался автоматически
mount | grep nvme
df -h /mnt/nvme
```

---

### Создание полезных директорий

```bash
# 22. Создаём структуру папок для хранения данных
mkdir -p /mnt/nvme/{backups,downloads,logs,opkg_cache}

# 23. Настраиваем логи на NVMe (опционально)
cat >> /etc/config/system << EOF

config system
    option log_file '/mnt/nvme/logs/system.log'
    option log_size '8192'
EOF

# 24. Перезапускаем логирование
/etc/init.d/log restart
```

---

### Проверка итогового результата

```bash
# 25. Финальная проверка
echo "=== Блочные устройства ===" && lsblk
echo "=== Монтирование ===" && mount | grep nvme
echo "=== Свободное место ===" && df -h /mnt/nvme
echo "=== Содержимое ===" && ls -la /mnt/nvme/
```

---

## 📝 Что нужно запомнить про OpenWrt 25.12.4:

| Вместо `opkg` | Используйте `apk` |
|---------------|-------------------|
| `opkg update` | `apk update` |
| `opkg install` | `apk add` |
| `opkg remove` | `apk del` |
| `opkg list-installed` | `apk info` |

**Ключевое отличие:** В 25.12.4 драйвер `kmod-nvme` НЕ входит в базовую прошивку, его нужно устанавливать отдельно через `apk add kmod-nvme`.

---

## 🎯 Итог

После выполнения всех шагов вы получите:
- **238.5 ГБ** дополнительного пространства в `/mnt/nvme`
- Автоматическое монтирование при загрузке
- Возможность хранить бэкапы, логи, загрузки и другие данные
- NVMe диск **не трогается** при обновлении прошивки

**Ваши данные в безопасности даже после `sysupgrade`!** 🚀

-------------------------------
<br/>



## Добавление NVMe диска в OpenWrt

---

### 📌 Для версии 24.10.5 (с пакетным менеджером `opkg`)

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

---

### 📌 Для версии 25.12.4 и новее (с пакетным менеджером `apk`)

**Важно:** В версии 25.12.4 драйвер `kmod-nvme` НЕ входит в базовую прошивку — его нужно установить отдельно!

```bash
# 1. Проверяем, увидел ли ядро NVMe контроллер
dmesg | grep -i nvme

# 2. Проверяем, виден ли диск на PCIe шине
cat /proc/bus/pci/devices | grep -i 1e4b

# 3. Смотрим, какие блочные устройства уже есть
lsblk

# 4. Обновляем списки пакетов (аналог opkg update)
apk update

# 5. Устанавливаем драйвер NVMe (КРИТИЧЕСКИ ВАЖНО!)
apk add kmod-nvme

# 6. Устанавливаем утилиты для работы с дисками и файловой системой
apk add fdisk gdisk e2fsprogs block-mount

# 7. Загружаем драйвер вручную (если не загрузился автоматически)
modprobe nvme

# 8. Проверяем, появился ли диск
lsblk
# Должны увидеть: nvme0n1 и nvme0n1p1 (если раздел уже был)

# 9. Создаём один primary-раздел на весь диск (если диск новый)
echo -e "o\nn\np\n1\n\n\nw" | fdisk /dev/nvme0n1

# 10. Форматируем раздел в ext4 с меткой
mkfs.ext4 -L nvme_data /dev/nvme0n1p1

# 11. Проверяем созданную файловую систему
blkid /dev/nvme0n1p1

# 12. Создаём точку монтирования
mkdir -p /mnt/nvme

# 13. Монтируем вручную для проверки
mount /dev/nvme0n1p1 /mnt/nvme

# 14. Проверяем, сколько места доступно
df -h /mnt/nvme

# 15. Генерируем шаблон fstab
block detect > /etc/config/fstab.new

# 16. Добавляем в основной конфиг
cat /etc/config/fstab.new >> /etc/config/fstab

# 17. Или настраиваем через UCI (рекомендуется)
uci set fstab.@mount[0]=mount
uci set fstab.@mount[0].device='/dev/nvme0n1p1'
uci set fstab.@mount[0].target='/mnt/nvme'
uci set fstab.@mount[0].fstype='ext4'
uci set fstab.@mount[0].options='rw,noatime'
uci set fstab.@mount[0].enabled='1'
uci set fstab.@mount[0].enabled_fsck='0'
uci commit fstab

# 18. Включаем и запускаем сервис монтирования
/etc/init.d/fstab enable
/etc/init.d/fstab restart

# 19. Проверяем, что диск примонтировался автоматически
mount | grep nvme
df -h /mnt/nvme

# 20. Создаём структуру папок для хранения данных
mkdir -p /mnt/nvme/{backups,downloads,logs,opkg_cache}

# 21. Финальная проверка
echo "=== Блочные устройства ===" && lsblk
echo "=== Монтирование ===" && mount | grep nvme
echo "=== Свободное место ===" && df -h /mnt/nvme
```

---

## 📊 Сравнение команд для разных версий

| Действие | OpenWrt 24.10.5 | OpenWrt 25.12.4+ |
|----------|-----------------|------------------|
| Обновить списки пакетов | `opkg update` | `apk update` |
| Установить драйвер NVMe | `opkg install kmod-nvme` | `apk add kmod-nvme` |
| Установить утилиты | `opkg install fdisk ...` | `apk add fdisk ...` |
| Загрузить драйвер | автоматически | `modprobe nvme` (может потребоваться) |
| Показать блочные устройства | `lsblk` (если установлен) или `cat /proc/partitions` | `lsblk` |

---

## ⚠️ Важные особенности версии 25.12.4+

1. **Драйвер NVMe нужно устанавливать отдельно** — команда `apk add kmod-nvme` обязательна
2. После установки драйвера может потребоваться `modprobe nvme` или перезагрузка
3. Менеджер пакетов `apk` вместо `opkg`
4. В остальном процесс идентичен

---

## 🎯 Для чего всё это

Теперь **до 256 ГБ** свободного места всегда доступны в `/mnt/nvme` – кладём туда:
- торренты
- бэкапы настроек
- Docker-контейнеры
- базы AdGuardHome
- Samba/NFS-шары
- логи системы
- кэш пакетов

**NVMe диск не трогается при обновлении прошивки!** 🚀


