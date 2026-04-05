# 📚 **Инструкция по установке Zapret2 на OpenWrt с NVMe диском**

## 📌 **Оглавление**
1. [Подготовка и установка NVMe диска](#1-подготовка-и-установка-nvme-диска)
2. [Скачивание и распаковка Zapret2](#2-скачивание-и-распаковка-zapret2)
3. [Создание структуры на NVMe](#3-создание-структуры-на-nvme)
4. [Установка зависимостей](#4-установка-зависимостей)
5. [Настройка Zapret2](#5-настройка-zapret2)
6. [Настройка DNS](#6-настройка-dns)
7. [Создание сервиса автозагрузки](#7-создание-сервиса-автозагрузки)
8. [Тестирование и проверка](#8-тестирование-и-проверка)

---

## 1️⃣ **Подготовка и установка NVMe диска**

### **Для чего:** NVMe диск используется для хранения программ и данных, чтобы не забивать встроенную флеш-память роутера (всего 200MB).

```bash
# 1. Проверяем, увидел ли диск
dmesg | grep -i nvme
# Вывод покажет: nvme0n1: 238.5 GB - диск обнаружен

# 2. Смотрим список NVMe-устройств
ls -l /dev/nvme*
# /dev/nvme0n1 - это весь диск
# /dev/nvme0n1p1 - будет раздел после форматирования

# 3. Обновляем список пакетов
opkg update
# Обновляет информацию о доступных пакетах из репозитория OpenWrt

# 4. Ставим утилиты для работы с дисками
opkg install fdisk gdisk e2fsprogs block-mount
# fdisk/gdisk - разметка диска
# e2fsprogs - создание файловых систем ext4
# block-mount - автоматическое монтирование

# 5. Создаём один primary-раздел на весь диск
echo -e "o\nn\np\n1\n\n\nw" | fdisk /dev/nvme0n1
# o - новая таблица разделов DOS
# n - новый раздел
# p - primary (основной)
# 1 - номер раздела
# \n\n - использовать весь диск
# w - записать изменения

# 6. Форматируем раздел в ext4 с меткой nvme_data
mkfs.ext4 -L nvme_data /dev/nvme0n1p1
# ext4 - надежная файловая система для Linux
# -L - задаем метку для удобства

# 7. Создаём точку монтирования
mkdir -p /mnt/nvme
# -p создает родительские директории при необходимости

# 8. Монтируем вручную для проверки
mount /dev/nvme0n1p1 /mnt/nvme

# 9. Проверяем доступное место
df -h /mnt/nvme
# Должно показать ~233 GB свободно

# 10. Настраиваем автоматическое монтирование при загрузке
block detect > /etc/config/fstab
# Генерирует шаблон конфигурации fstab

# 11-17. Настраиваем fstab через UCI
uci set fstab.@mount[0].device='/dev/nvme0n1p1'  # Указываем устройство
uci set fstab.@mount[0].target='/mnt/nvme'       # Точка монтирования
uci set fstab.@mount[0].fstype='ext4'            # Тип ФС
uci set fstab.@mount[0].options='rw,noatime'     # rw-чтение/запись, noatime-не обновлять время доступа
uci set fstab.@mount[0].enabled='1'              # Включаем монтирование
uci set fstab.@mount[0].enabled_fsck='0'         # Отключаем проверку ФС при загрузке
uci commit fstab                                  # Сохраняем изменения

# 18. Включаем сервис монтирования
/etc/init.d/fstab enable   # Добавляем в автозагрузку
/etc/init.d/fstab restart  # Запускаем сейчас

# 19. Проверяем итоговую таблицу
df -h
# /dev/nvme0n1p1 должен быть смонтирован в /mnt/nvme
```

---

## 2️⃣ **Скачивание и распаковка Zapret2**

### **Для чего:** Zapret2 - программа для обхода DPI (Deep Packet Inspection) блокировок.

```bash
# Переходим в папку загрузок на NVMe
cd /mnt/nvme/downloads
# Создаем папку, если её нет
mkdir -p /mnt/nvme/downloads

# Скачиваем архив с программой (ВАЖНО: используем -O для правильного имени)
wget -O zapret2-v0.9.4.7-openwrt-embedded.tar.gz \
  "https://github.com/bol-van/zapret2/releases/download/v0.9.4.7/zapret2-v0.9.4.7-openwrt-embedded.tar.gz"
# -O указывает имя выходного файла
# Без -O GitHub может сохранить под временным именем

# Распаковываем архив
tar -xzvf zapret2-v0.9.4.7-openwrt-embedded.tar.gz
# x - extract (распаковать)
# z - gzip (архив сжат)
# v - verbose (показывать процесс)
# f - file (имя файла)

# Смотрим содержимое
ls -la zapret2-v0.9.4.7/
```

---

## 3️⃣ **Создание структуры на NVMe**

### **Для чего:** Организуем правильную структуру директорий для работы Zapret2.

```bash
# 1. Создаем основную папку для Zapret на NVMe
mkdir -p /mnt/nvme/zapret
# -p создает все промежуточные директории

# 2. Копируем все файлы из распакованного архива
cp -r /mnt/nvme/downloads/zapret2-v0.9.4.7/* /mnt/nvme/zapret/
# -r рекурсивное копирование всех поддиректорий

# 3. Создаем папку для бинарников nfqws2
mkdir -p /mnt/nvme/zapret/nfq2

# 4. Копируем бинарные файлы для архитектуры aarch64 (ARM64)
cp /mnt/nvme/zapret/binaries/linux-arm64/nfqws2 /mnt/nvme/zapret/nfq2/
cp /mnt/nvme/zapret/binaries/linux-arm64/ip2net /mnt/nvme/zapret/
cp /mnt/nvme/zapret/binaries/linux-arm64/mdig /mnt/nvme/zapret/

# 5. Делаем бинарники исполняемыми
chmod +x /mnt/nvme/zapret/nfq2/nfqws2
chmod +x /mnt/nvme/zapret/ip2net
chmod +x /mnt/nvme/zapret/mdig

# 6. Создаем симлинк /opt/zapret2 -> /mnt/nvme/zapret
mkdir -p /opt
ln -sf /mnt/nvme/zapret /opt/zapret2
# ln -s создает символическую ссылку
# -f перезаписывает существующую

# 7. Проверяем структуру
ls -la /opt/zapret2/
# Должны увидеть все скопированные файлы
```

---

## 4️⃣ **Установка зависимостей**

### **Для чего:** Zapret2 требует дополнительные модули ядра и утилиты для работы с сетевыми пакетами.

```bash
# Устанавливаем необходимые пакеты
opkg install iptables iptables-mod-filter iptables-mod-nfqueue \
  iptables-mod-u32 kmod-ipt-nfqueue libnetfilter-queue1

# Что устанавливаем:
# iptables - система фильтрации пакетов
# iptables-mod-nfqueue - модуль для очередей netfilter
# kmod-ipt-nfqueue - модуль ядра для NFQUEUE
# libnetfilter-queue1 - библиотека для работы с очередями

# Проверяем загрузку модуля ядра
modprobe nfnetlink_queue
# Загружает модуль в ядро для работы с очередями

# Проверяем, что модуль загружен
lsmod | grep nfnetlink
```

---

## 5️⃣ **Настройка Zapret2**

### **Для чего:** Настраиваем параметры обхода DPI для HTTP и HTTPS трафика.

```bash
# 1. Создаем конфигурационный файл (найденный через blockcheck2.sh)
cat > /mnt/nvme/zapret/config << 'EOF'
# Zapret config for OpenWrt
NFQWS2_ENABLE=1
NFQWS2_PORTS_TCP=443,80
DISABLE_IPV6=1
INIT_APPLY_FW=1
WS_USER=nobody
DESYNC_MARK=0x40000000
EOF

# 2. Создаем файл функций для init скрипта
cat > /mnt/nvme/zapret/init.d/functions << 'EOF'
#!/bin/sh

standard_mode_daemons() {
    return 0
}

custom_runner() {
    return 0
}

zapret_apply_firewall() {
    iptables -t mangle -I PREROUTING -p tcp --dport 443 -j NFQUEUE --queue-num 0
    iptables -t mangle -I OUTPUT -p tcp --dport 443 -j NFQUEUE --queue-num 0
    iptables -t mangle -I PREROUTING -p tcp --dport 80 -j NFQUEUE --queue-num 1
    iptables -t mangle -I OUTPUT -p tcp --dport 80 -j NFQUEUE --queue-num 1
}

zapret_unapply_firewall() {
    iptables -t mangle -D PREROUTING -p tcp --dport 443 -j NFQUEUE --queue-num 0
    iptables -t mangle -D OUTPUT -p tcp --dport 443 -j NFQUEUE --queue-num 0
    iptables -t mangle -D PREROUTING -p tcp --dport 80 -j NFQUEUE --queue-num 1
    iptables -t mangle -D OUTPUT -p tcp --dport 80 -j NFQUEUE --queue-num 1
}

linux_fwtype() {
    echo "iptables"
}

openwrt_fw3_integration() {
    return 1
}
EOF

chmod +x /mnt/nvme/zapret/init.d/functions
```

---

## 6️⃣ **Настройка DNS**

### **Для чего:** Правильные DNS серверы необходимы для корректного резолвинга заблокированных доменов.

```bash
# 1. Настраиваем DNS через UCI
uci set dhcp.@dnsmasq[0].server='8.8.8.8'
uci add_list dhcp.@dnsmasq[0].server='1.1.1.1'
uci set dhcp.@dnsmasq[0].noresolv='1'
uci set dhcp.@dnsmasq[0].localuse='0'
uci commit dhcp

# 2. Принудительно создаем resolv.conf
echo "nameserver 8.8.8.8" > /tmp/resolv.conf
echo "nameserver 1.1.1.1" >> /tmp/resolv.conf

# 3. Перезапускаем DNS сервер
/etc/init.d/dnsmasq restart

# 4. Проверяем
nslookup rutracker.org 8.8.8.8
# Должен показать IP-адреса: 104.21.32.39 и 172.67.182.196
```

---

## 7️⃣ **Создание сервиса автозагрузки**

### **Для чего:** Автоматический запуск Zapret при загрузке роутера.

```bash
# Создаем init скрипт
cat > /etc/init.d/zapret << 'EOF'
#!/bin/sh /etc/rc.common

START=95
STOP=10

start() {
    echo "Starting Zapret (HTTP + HTTPS with separate queues)..."
    
    # Загружаем модуль ядра для работы с очередями
    modprobe nfnetlink_queue 2>/dev/null
    
    # Очищаем старые правила и процессы
    killall nfqws2 2>/dev/null
    iptables -t mangle -F 2>/dev/null
    
    # Запускаем обработчик HTTPS (порт 443, очередь 0)
    /opt/zapret2/nfq2/nfqws2 --daemon --qnum=0 \
        --lua-init=@/opt/zapret2/lua/zapret-lib.lua \
        --lua-init=@/opt/zapret2/lua/zapret-antidpi.lua \
        --filter-tcp=443 \
        --lua-desync=fake:blob=fake_default_tls:tcp_ts=-1000
    
    sleep 1
    
    # Запускаем обработчик HTTP (порт 80, очередь 1)
    /opt/zapret2/nfq2/nfqws2 --daemon --qnum=1 \
        --lua-init=@/opt/zapret2/lua/zapret-lib.lua \
        --lua-init=@/opt/zapret2/lua/zapret-antidpi.lua \
        --filter-tcp=80 \
        --lua-desync=http_methodeol
    
    # Настраиваем правила iptables для перенаправления трафика
    iptables -t mangle -I PREROUTING 1 -p tcp --dport 443 -j NFQUEUE --queue-num 0
    iptables -t mangle -I OUTPUT 1 -p tcp --dport 443 -j NFQUEUE --queue-num 0
    iptables -t mangle -I PREROUTING 1 -p tcp --dport 80 -j NFQUEUE --queue-num 1
    iptables -t mangle -I OUTPUT 1 -p tcp --dport 80 -j NFQUEUE --queue-num 1
    
    sleep 2
    
    local count=$(pgrep nfqws2 | wc -l)
    echo "✓ Zapret started ($count instance(s))"
}

stop() {
    echo "Stopping Zapret..."
    killall nfqws2 2>/dev/null
    iptables -t mangle -F 2>/dev/null
}

status() {
    local count=$(pgrep nfqws2 | wc -l)
    if [ $count -gt 0 ]; then
        echo "Zapret running ($count instance(s))"
        iptables -t mangle -L -n -v 2>/dev/null | grep -E "(Chain|NFQUEUE)"
    else
        echo "Zapret stopped"
    fi
}

restart() { stop; sleep 1; start; }

EOF

# Делаем скрипт исполняемым
chmod +x /etc/init.d/zapret

# Добавляем в автозагрузку
/etc/init.d/zapret enable

# Запускаем сейчас
/etc/init.d/zapret start
```

---

## 8️⃣ **Тестирование и проверка**

### **Для чего:** Убедиться, что всё работает корректно.

```bash
# 1. Проверяем статус сервиса
/etc/init.d/zapret status
# Должен показать "Zapret running (2 instance(s))"

# 2. Проверяем процессы
ps | grep nfqws2
# Должны увидеть 2 процесса: для очередей 0 и 1

# 3. Проверяем правила iptables
iptables -t mangle -L -n -v | grep NFQUEUE
# Должны увидеть правила для портов 80 и 443

# 4. Тестируем доступ к сайтам
curl -I --max-time 15 https://rutracker.org
curl -I --max-time 15 https://www.youtube.com/
curl -I --max-time 15 https://google.com

# 5. Проверяем DNS
nslookup rutracker.org
nslookup youtube.com

# 6. Смотрим статистику пакетов
watch -n 1 'iptables -t mangle -L -n -v | grep NFQUEUE'
# Нажмите Ctrl+C для выхода
```

---

## 📊 **Итоговая структура**

```
/mnt/nvme/
├── downloads/          # Временные файлы и архивы
│   └── zapret2-v0.9.4.7/  # Распакованный архив
└── zapret/            # Основная установка Zapret
    ├── nfq2/
    │   └── nfqws2     # Основной бинарник
    ├── lua/           # Lua скрипты для обхода DPI
    ├── init.d/        # Скрипты инициализации
    └── config         # Конфигурационный файл

/etc/init.d/
└── zapret            # Init скрипт для автозагрузки

/opt/
└── zapret2 -> /mnt/nvme/zapret   # Симлинк для удобства
```

---

## 🔧 **Полезные команды для управления**

```bash
# Запуск/остановка/перезапуск
/etc/init.d/zapret start
/etc/init.d/zapret stop
/etc/init.d/zapret restart

# Статус
/etc/init.d/zapret status

# Включение/отключение автозагрузки
/etc/init.d/zapret enable
/etc/init.d/zapret disable

# Просмотр логов
logread | grep -i zapret

# Просмотр статистики NFQUEUE
iptables -t mangle -L -n -v | grep NFQUEUE

# Мониторинг в реальном времени
watch -n 1 'iptables -t mangle -L -n -v | grep -E "(Chain|NFQUEUE)"'
```

---

## ⚠️ **Важные замечания**

1. **Архитектура:** Инструкция написана для **aarch64** (ARM64). Для других архитектур меняйте путь в `binaries/linux-*`

2. **NVMe диск:** Все данные хранятся на NVMe, что экономит встроенную флеш-память роутера

3. **Обход DPI:** Используются параметры, найденные через `blockcheck2.sh` специально для rutracker.org

4. **Две очереди:** HTTP и HTTPS обрабатываются отдельными экземплярами с разными очередями NFQUEUE (0 и 1)

5. **DNS:** Принудительно установлены DNS серверы Google (8.8.8.8) и Cloudflare (1.1.1.1)

---





