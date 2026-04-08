# 📚 Полная инструкция по обновлению Banana Pi OpenWrt Router EU (OpenWrt One/BPi-R4)
См. офф. док.:
- [Upgrading OpenWrt firmware using CLI](https://openwrt.org/docs/guide-user/installation/sysupgrade.cli)
- [Upgrading OpenWrt firmware using LuCI and CLI](https://openwrt.org/docs/guide-user/installation/generic.sysupgrade)

## 🎯 **Цель обновления**
Переход с проблемной **SNAPSHOT версии OpenWrt** на стабильную **OpenWrt 24.10.5** для решения проблем с репозиториями пакетов.
<br/>


## 🔍 **Шаг 1: Диагностика исходной системы**

### **Команды диагностики:**
```bash
# Проверка текущей версии
cat /etc/openwrt_release

# Проверка архитектуры
uname -a
cat /proc/cpuinfo

# Проверка сетевых интерфейсов
ip addr show
ip route show

# Проверка установленных пакетов
opkg list-installed

# Проверка репозиториев
cat /etc/opkg/distfeeds.conf
```

### **Исходные данные:**
- **Версия:** OpenWrt SNAPSHOT r27876+1-3098b4bf07
- **Архитектура:** aarch64_cortex-a53
- **Target:** mediatek/filogic
- **Устройство:** OpenWrt One (коммерчески Banana Pi OpenWrt Router EU)
- **Проблема:** Не работал `opkg update` (ошибки SSL/репозиториев)
<br/>


## 🌐 **Шаг 2: Поиск правильной прошивки**

### **Читаем/ищем:**
- [Загрузить прошивку OpenWrt для вашего устройства](https://firmware-selector.openwrt.org/?version=24.10.5&target=mediatek%2Ffilogic&id=bananapi_bpi-r4)
- [Mirrors with firmwares](https://mirror-03.infra.openwrt.org/releases/24.10.5/targets/mediatek/filogic/)
- [downloads.openwrt.org/releases/24.10.5/targets/mediatek/filogic/](https://downloads.openwrt.org/releases/24.10.5/targets/mediatek/filogic/)
- [downloads.openwrt.org/releases/25.12.2/targets/mediatek/filogic/](https://downloads.openwrt.org/releases/25.12.2/targets/mediatek/filogic/)

### **Ключевые ресурсы:**
1. **Firmware Selector:** https://firmware-selector.openwrt.org
   - Параметры: version=24.10.5, target=mediatek/filogic
   - ОШИБКА: Искали для `bananapi_bpi-r4`, а нужно для `openwrt_one`

2. **Репозитории OpenWrt:** https://downloads.openwrt.org/releases/24.10.5/targets/mediatek/filogic/
   - Просмотр через браузер или wget
   - Ключевое открытие: файлы с **`openwrt_one-*`** а не **`bananapi_bpi-r4-*`**

3. **Официальная документация:**
   - [OpenWrt One](https://openwrt.org/toh/openwrt/one)
   - [Quick start guide for OpenWrt installation](https://openwrt.org/docs/guide-quick-start/start)
   - [Sinovoip BananaPi BPI-R4](https://openwrt.org/inbox/toh/sinovoip/bananapi_bpi-r4)
<br/>


## 📥 **Шаг 3: Скачивание прошивки**

### **Ошибочные файлы (не подошли):**
```bash
# Неправильные (для bananapi_bpi-r4)
openwrt-24.10.5-mediatek-filogic-bananapi_bpi-r4-squashfs-sysupgrade.itb
openwrt-24.10.5-mediatek-filogic-bananapi_bpi-r4-sdcard.img.gz
```

### **Правильные файлы (для openwrt_one):**
```bash
# Основная прошивка
openwrt-24.10.5-mediatek-filogic-openwrt_one-squashfs-sysupgrade.itb

# Recovery образ
openwrt-24.10.5-mediatek-filogic-openwrt_one-initramfs.itb
```

### **Команды скачивания:**
```bash
# В recovery режиме
cd /tmp
wget https://downloads.openwrt.org/releases/24.10.5/targets/mediatek/filogic/openwrt-24.10.5-mediatek-filogic-openwrt_one-squashfs-sysupgrade.itb
wget https://downloads.openwrt.org/releases/24.10.5/targets/mediatek/filogic/openwrt-24.10.5-mediatek-filogic-openwrt_one-initramfs.itb
```
<br/>


## 🔄 **Шаг 4: Процесс прошивки**

### **Попытки и ошибки:**
```bash
# 1. Неудачная попытка (неправильный файл)
sysupgrade -n openwrt-24.10.5-mediatek-filogic-bananapi_bpi-r4-squashfs-sysupgrade.itb
# Ошибка: "Device openwrt,one not supported by this image"

# 2. Принудительная прошивка (перезагрузка в recovery)
sysupgrade -n -F openwrt-24.10.5-mediatek-filogic-bananapi_bpi-r4-squashfs-sysupgrade.itb
# Результат: Загрузка в recovery (initramfs) режим

# 3. Успешная прошивка из recovery
sysupgrade -n openwrt-24.10.5-mediatek-filogic-openwrt_one-squashfs-sysupgrade.itb
# УСПЕХ: Установлена OpenWrt 24.10.5
```
<br/>


## ⚙️ **Шаг 5: Послеустановочная настройка**

### **Настройка сети:**
```bash
# PPPoE настройка
uci set network.wan.proto='pppoe'
uci set network.wan.username='ваш_логин'
uci set network.wan.password='ваш_пароль'
uci set network.wan.device='eth0'
uci commit network
/etc/init.d/network restart
```

### **Настройка репозиториев:**
```bash
# Автоматически настроены в 24.10.5:
cat /etc/opkg/distfeeds.conf
# Содержит корректные ссылки для aarch64_cortex-a53
```
```conf
src/gz openwrt_core https://downloads.openwrt.org/releases/24.10.5/targets/mediatek/filogic/packages
src/gz openwrt_base https://downloads.openwrt.org/releases/24.10.5/packages/aarch64_cortex-a53/base
src/gz openwrt_kmods https://downloads.openwrt.org/releases/24.10.5/targets/mediatek/filogic/kmods/6.6.119-1-6a9e125268c43e0bae8cecb014c8ab03
src/gz openwrt_luci https://downloads.openwrt.org/releases/24.10.5/packages/aarch64_cortex-a53/luci
src/gz openwrt_packages https://downloads.openwrt.org/releases/24.10.5/packages/aarch64_cortex-a53/packages
src/gz openwrt_routing https://downloads.openwrt.org/releases/24.10.5/packages/aarch64_cortex-a53/routing
src/gz openwrt_telephony https://downloads.openwrt.org/releases/24.10.5/packages/aarch64_cortex-a53/telephony
```

### **Установка пакетов:**
```bash
# Обновление списков
opkg update

# Установка базовых утилит
opkg install mc nano curl htop

# Проверка
nano --version
curl --version
```
<br/>


## 🛠 **Шаг 6: Создание SSH ключей (опционально)**

### **На роутере:**
```bash
mkdir -p ~/.ssh
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -C "r4_banana" -N ""
cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

### **На Windows (PowerShell):**
```powershell
ssh-keygen -t rsa -b 4096
# Ключи сохраняются в: C:\Users\ваш_пользователь\.ssh\
```
<br/>


## 📋 **Итоговая последовательность команд:**

1. **Диагностика:** `cat /etc/openwrt_release`
2. **Скачивание:** `wget http://.../openwrt_one-squashfs-sysupgrade.itb`
3. **Прошивка:** `sysupgrade -n openwrt_one-squashfs-sysupgrade.itb`
4. **Настройка PPPoE:** через UCI или веб-интерфейс
5. **Обновление:** `opkg update`
6. **Установка:** `opkg install nano curl htop`
7. **Пароль:** `passwd`
<br/>

--------------------------

# 📖 СИСТЕМА КОМАНД OPENWRT

## 📦 **Управление пакетами (opkg)**
```bash
# Основные команды
opkg update                    # Обновить списки пакетов
opkg install <пакет>          # Установить пакет
opkg remove <пакет>           # Удалить пакет
opkg list                     # Показать все пакеты
opkg list-installed          # Показать установленные
opkg list-upgradable         # Показать доступные обновления
opkg upgrade <пакет>         # Обновить пакет
opkg --force-overwrite upgrade luci
opkg info <пакет>            # Информация о пакете

# Поиск
opkg list | grep -i nano     # Поиск пакета
opkg find "*nano*"           # Альтернативный поиск
```

## 🌐 **Сетевые команды**
```bash
# Интерфейсы
ip addr show                 # Показать все интерфейсы
ip link show                 # Показать состояние интерфейсов
ifconfig                    # Альтернатива (устаревшая)
ip route show               # Показать таблицу маршрутизации

# DNS и сеть
nslookup google.com         # Проверить DNS
ping -c 3 8.8.8.8          # Проверить соединение
traceroute google.com       # Трассировка маршрута

# Wi-Fi
iwinfo                      # Информация о Wi-Fi
iwinfo wlan0 scan          # Сканировать сети
wifi                        # Управление Wi-Fi
wifi up/down               # Включить/выключить Wi-Fi
```

## ⚙️ **Конфигурация (UCI - Unified Configuration Interface)**
```bash
# Основные команды
uci show                    # Показать всю конфигурацию
uci show network           # Показать конфиг сети
uci show wireless          # Показать конфиг Wi-Fi

# Изменение конфигурации
uci set network.wan.proto='pppoe'     # Изменить значение
uci get network.wan.proto            # Получить значение
uci commit network                   # Сохранить изменения
uci changes                         # Показать неподтвержденные изменения
uci revert network                  # Отменить изменения

# Перезапуск служб
/etc/init.d/network restart        # Перезапуск сети
/etc/init.d/firewall restart       # Перезапуск фаервола
/etc/init.d/dnsmasq restart        # Перезапуск DNS/DHCP
```

## 🖥 **Системные команды**
```bash
# Процессы
ps                           # Показать процессы
top                         # Динамический просмотр процессов
htop                        # Расширенный top (если установлен)
kill <PID>                  # Завершить процесс
killall <имя>               # Завершить все процессы с именем

# Диски и память
df -h                       # Свободное место на дисках
free -h                     # Использование памяти
du -sh <папка>              # Размер папки

# Системная информация
uname -a                    # Информация о системе
cat /proc/cpuinfo           # Информация о процессоре
cat /proc/meminfo           # Информация о памяти
cat /etc/openwrt_release    # Версия OpenWrt
logread                     # Просмотр логов
dmesg                       # Сообщения ядра
```

## 🔧 **Управление службами**
```bash
# Статус служб
/etc/init.d/<служба> status     # Статус службы
service --list-all              # Все службы

# Управление
/etc/init.d/<служба> start      # Запустить
/etc/init.d/<служба> stop       # Остановить
/etc/init.d/<служба> restart    # Перезапустить
/etc/init.d/<служба> enable     # Включить автозагрузку
/etc/init.d/<служба> disable    # Отключить автозагрузку
```

## 📁 **Работа с файлами**
```bash
# Просмотр и редактирование
cat <файл>                     # Показать файл
less <файл>                    # Просмотр с прокруткой
nano <файл>                    # Редактирование (если установлен)
vi <файл>                      # Редактирование (встроенный)

# Поиск
find / -name "*.conf"          # Найти файлы
grep -r "текст" /etc/         # Рекурсивный поиск текста

# Права и владельцы
chmod 755 <файл>               # Изменить права
chown root:root <файл>         # Изменить владельца
```

## 🛡 **Безопасность**
```bash
# Пароли
passwd                         # Сменить пароль root
echo -e "пароль\nпароль" | passwd root  # Установить пароль без диалога

# SSH
ssh-keygen                     # Генерация ключей
dropbearkey -t rsa -f /etc/dropbear/dropbear_rsa_host_key  # Ключ SSH сервера

# Фаервол
fw4 print                      # Показать правила фаервола
fw4 restart                    # Перезапустить фаервол
```

## 🔄 **Системные операции**
```bash
# Перезагрузка и выключение
reboot                        # Перезагрузка
poweroff                      # Выключение

# Бэкап и восстановление
sysupgrade -b backup.tar.gz   # Создать бэкап
sysupgrade -r backup.tar.gz   # Восстановить из бэкапа

# Время и дата
date                          # Текущая дата и время
date -s "2024-12-28 12:00:00" # Установить время
```

## 📊 **Мониторинг**
```bash
# Сетевой мониторинг
iftop                        # Потребление трафика (если установлен)
bmon                         # Мониторинг интерфейсов
vnstat                       # Статистика трафика

# Системный мониторинг
collectd                     # Система сбора статистики
logread -f                   # Просмотр логов в реальном времени
```

## 🚀 **Полезные команды для Banana Pi R4**
```bash
# Аппаратная информация
cat /proc/device-tree/model   # Модель устройства
cat /proc/mtd                 # Информация о флеш-памяти

# Терминал
screen                        # Менеджер терминалов
tmux                          # Альтернативный менеджер (если установлен)

# Обновление системы
sysupgrade -n <файл>         # Прошивка без сохранения настроек
sysupgrade -c <файл>         # Проверить файл прошивки
```


## 💡 **Важные особенности OpenWrt на Banana Pi R4:**

1. **Файловая система:** SquashFS + Overlay (безопасная, позволяет сброс)
2. **Конфигурация:** Хранится в `/etc/config/`
3. **Логи:** Просматривать через `logread`
4. **Сеть:** Основные интерфейсы `eth0` (WAN), `eth1` (LAN), `wlan0/1` (Wi-Fi)
5. **Wi-Fi:** Два радио: 2.4GHz (phy0) и 5GHz (phy1)
6. **Прошивка:** Только файлы с `openwrt_one-*` в названии


--------------------------


### 📋 Общая структура и политики  firewall
*   **Имя таблицы**: `table inet fw4` (обрабатывает IPv4 и IPv6 одновременно).
*   **Политики по умолчанию** (самые важные строчки):
    *   **Цепочка `input`**: `policy drop`. **Блокирует все ВХОДЯЩИЕ** соединения извне (из интернета) к самому роутеру, если нет специального разрешающего правила. Это основная защита.
    *   **Цепочка `forward`**: `policy drop`. **Блокирует весь ПРОХОДЯЩИЙ** трафик (между сетями, например, из интернета в вашу локальную сеть).
    *   **Цепочка `output`**: `policy accept`. **Разрешает все ИСХОДЯЩИЕ** соединения от роутера (например, запросы во внешний DNS).

**Вывод:** Базовая настройка безопасна — роутер "молчит" на все входящие запросы из WAN.

---

### 🔀 Логика работы цепочек (как трафик проходит)
Вот упрощённая схема, куда направляется трафик:

```mermaid
flowchart TD
    A[ВХОДЯЩИЙ трафик<br>из Интернета (WAN)] --> B{Какой интерфейс?}
    
    B -->|"pppoe-wan или eth0"| C["Цепочка input_wan<br>(Строгая проверка)"]
    C --> D["Пропускает только служебный трафик<br>(Ping, DHCP, ICMPv6)"]
    D --> E[ВСЁ ОСТАЛЬНОЕ ОТБРАСЫВАЕТСЯ]
    
    B -->|br-lan<br>(локальная сеть)| F["Цепочка input_lan<br>(Разрешающая)"]
    F --> G[ПРАКТИЧЕСКИ ВСЁ РАЗРЕШАЕТСЯ<br>к роутеру]
```

```mermaid
flowchart LR
    H[ПРОХОДЯЩИЙ трафик<br>(трансляция между сетями)] --> I{Направление?}
    
    I -->|"ИЗ локальной сети (LAN)<br>В интернет (WAN)"| J["Цепочка forward_lan -><br>accept_to_wan (РАЗРЕШЕНО)"]
    J --> K[Трафик проходит с трансляцией адресов (MASQUERADE)]
    
    I -->|"ИЗ интернета (WAN)<br>В локальную сеть (LAN)"| L["Цепочка forward_wan -><br>reject_to_wan (БЛОКИРОВКА)"]
    L --> M[Трафик блокируется.<br>Для проброса портов нужны правила.]
```

---

### 🔧 Ключевые зоны и интерфейсы (как у вас настроено)
В вашем конфиге определены зоны, что очень помогает понять логику:

| Зона | Интерфейсы в зоне | Назначение |
| :--- | :--- | :--- |
| **`lan`** | `br-lan` | Ваша внутренняя сеть (объединение Ethernet и Wi-Fi). Доверенная. |
| **`wan`** | `pppoe-wan`, `eth0` | Внешний интернет. `pppoe-wan` — PPPoE-соединение, `eth0` — физический WAN-порт. Недоверенная. |

---

### ✅ Что **РАЗРЕШЕНО** по умолчанию (без ваших правок)
1.  **Из LAN в интернет (WAN)**: Всё. Это основная функция роутера.
2.  **Из LAN к самому роутеру**: Практически все службы (SSH, веб-интерфейс и т.д.).
3.  **Основные сетевые протоколы из WAN к роутеру** (только для его работы):
    *   **Ответы на Ping** (`icmp type 8`) — можно проверить доступность роутера извне.
    *   **DHCP-клиент** (`udp dport 68`) — для получения IP от провайдера.
    *   **IGMP** — для IPTV (часто нужно).
    *   **ICMPv6** — обязателен для работы IPv6.

4.  **Трансляция адресов (NAT/Masquerade)**: Автоматически настроена в цепочке `srcnat_wan`. Все ваши устройства в LAN выходят в интернет под одним IP-адресом роутера.

---

### ❌ Что **ЗАБЛОКИРОВАНО** по умолчанию (самая важная защита)
1.  **Любые входящие соединения из интернета (WAN) к службам роутера** (кроме пункта 3 выше). Веб-интерфейс, SSH и т.д. **недоступны извне**.
2.  **Любые входящие соединения из интернета (WAN) в вашу локальную сеть (LAN)**. Это защищает ваши компьютеры и устройства.

---

### 🛠️ Как вносить изменения (практические команды)
Не редактируйте файлы вручную! Используйте `uci` (Unified Configuration Interface):

| Задача | Команда UCI |
| :--- | :--- |
| **Открыть порт из интернета на роутер** (например, для внешнего SSH) | `uci add firewall rule` <br> `uci set firewall.@rule[-1].name='Allow-WAN-SSH'` <br> `uci set firewall.@rule[-1].src='wan'` <br> `uci set firewall.@rule[-1].proto='tcp'` <br> `uci set firewall.@rule[-1].dest_port='22'` <br> `uci set firewall.@rule[-1].target='ACCEPT'` <br> `uci commit firewall` <br> `service firewall restart` |
| **Пробросить порт из интернета на устройство в LAN** (Port Forwarding) | `uci add firewall redirect` <br> `uci set firewall.@redirect[-1].name='Forward-WEB'` <br> `uci set firewall.@redirect[-1].src='wan'` <br> `uci set firewall.@redirect[-1].proto='tcp'` <br> `uci set firewall.@redirect[-1].src_dport='80'` <br> `uci set firewall.@redirect[-1].dest_ip='192.168.1.100'` <br> `uci set firewall.@redirect[-1].dest_port='80'` <br> `uci commit firewall` <br> `service firewall restart` |
| **Посмотреть все пользовательские правила** | `uci show firewall` |
| **Включить/выключить фаервол** | `/etc/init.d/firewall enable/disable` <br> `/etc/init.d/firewall start/stop/restart` |

---

Отлично! Вот полный набор команд для управления правилами фаервола OpenWrt (fw4) **через UCI** и **прямо в nftables**.

## 📦 **1. УПРАВЛЕНИЕ ЧЕРЕЗ UCI (Рекомендуемый способ)**

### 🔍 **Просмотр существующих правил**
```bash
# Показать ВСЮ конфигурацию фаервола
uci show firewall

# Показать только пользовательские правила
uci show firewall | grep "firewall.@rule" | head -20

# Показать только пробросы портов
uci show firewall | grep "firewall.@redirect" | head -20

# Показать зоны и сети
uci show firewall.zone
```

### ➕ **Добавление новых правил**

#### **A. Разрешить доступ к роутеру из WAN (например, SSH)**
```bash
# 1. Создаем правило
uci add firewall rule

# 2. Настраиваем параметры
uci set firewall.@rule[-1].name='Allow-WAN-SSH'
uci set firewall.@rule[-1].src='wan'          # Источник: интернет
uci set firewall.@rule[-1].dest='lan'         # Назначение: роутер
uci set firewall.@rule[-1].proto='tcp'        # Протокол TCP
uci set firewall.@rule[-1].dest_port='22'     # Порт назначения: 22 (SSH)
uci set firewall.@rule[-1].family='ipv4'      # Только IPv4
uci set firewall.@rule[-1].target='ACCEPT'    # Действие: разрешить

# 3. Сохраняем и применяем
uci commit firewall
service firewall restart
```

#### **B. Пробросить порт из интернета на устройство в LAN**
```bash
# Проброс порта 80 на веб-сервер в LAN
uci add firewall redirect

uci set firewall.@redirect[-1].name='Forward-WEB-Server'
uci set firewall.@redirect[-1].src='wan'
uci set firewall.@redirect[-1].proto='tcp'
uci set firewall.@redirect[-1].src_dport='80'        # Внешний порт
uci set firewall.@redirect[-1].dest_ip='192.168.1.100' # IP в LAN
uci set firewall.@redirect[-1].dest_port='80'        # Внутренний порт
uci set firewall.@redirect[-1].dest='lan'

uci commit firewall
service firewall restart
```

#### **C. Разрешить доступ к определенной службе только из одной подсети в LAN**
```bash
# Разрешить HTTP только с 192.168.1.0/24 к маршрутизатору
uci add firewall rule

uci set firewall.@rule[-1].name='Allow-LAN-Subnet-HTTP'
uci set firewall.@rule[-1].src='lan'
uci set firewall.@rule[-1].src_ip='192.168.1.0/24'  # Только эта подсеть
uci set firewall.@rule[-1].dest='lan'               # На сам роутер
uci set firewall.@rule[-1].proto='tcp'
uci set firewall.@rule[-1].dest_port='80'
uci set firewall.@rule[-1].target='ACCEPT'

uci commit firewall
service firewall restart
```

### ✏️ **Редактирование существующих правил**
```bash
# 1. Найдите ID правила (обратите внимание на число в квадратных скобках)
uci show firewall | grep -E "@rule\[[0-9]+\]\.name"

# Пример вывода: firewall.@rule[2].name='Allow-WAN-SSH'

# 2. Измените нужный параметр
uci set firewall.@rule[2].dest_port='2222'  # Меняем порт с 22 на 2222
uci set firewall.@rule[2].enabled='0'      # Временно отключаем правило (0=выкл, 1=вкл)

uci commit firewall
service firewall restart
```

### ❌ **Удаление правил**
```bash
# 1. Найдите ID правила для удаления
uci show firewall | grep -n "Allow-WAN-SSH"

# 2. Удалите правило по ID
uci delete firewall.@rule[2]  # Где 2 - номер из вывода выше

# ИЛИ удалите по имени (более безопасно)
uci delete $(uci show firewall | grep -B1 "Allow-WAN-SSH" | grep "@rule" | cut -d= -f1)

uci commit firewall
service firewall restart
```

### 🔄 **Временное отключение/включение правил**
```bash
# Отключить правило (остается в конфиге)
uci set firewall.@rule[2].enabled='0'

# Включить правило
uci set firewall.@rule[2].enabled='1'

uci commit firewall
service firewall restart
```

## ⚡ **2. ПРЯМОЕ УПРАВЛЕНИЕ NFTABLES (Для опытных)**

### 🔍 **Просмотр текущих правил nftables**
```bash
# Показать ВСЕ правила (аналогично fw4 print, но более подробно)
nft list ruleset

# Показать только правила фильтрации
nft list table inet fw4

# Показать только цепочку input
nft list chain inet fw4 input

# Показать счётчики (сколько раз сработало правило)
nft list ruleset -a | grep counter
```

### ➕ **Добавление временных правил (до перезагрузки)**
```bash
# Разрешить HTTPS (443) из WAN на роутер
nft add rule inet fw4 input iifname { "pppoe-wan", "eth0" } tcp dport 443 ct state new counter accept

# Разрешить диапазон портов для игр
nft add rule inet fw4 input iifname { "pppoe-wan", "eth0" } tcp dport { 27015-27030 } ct state new counter accept

# Разрешить только с определенного IP
nft add rule inet fw4 input iifname { "pppoe-wan", "eth0" } ip saddr 93.184.216.34 tcp dport 22 ct state new counter accept
```

### ❌ **Удаление временных правил**
```bash
# 1. Найдите handle правила
nft list chain inet fw4 input -a

# Пример вывода: ... handle 28 tcp dport 443 ct state new counter accept

# 2. Удалите по handle
nft delete rule inet fw4 input handle 28
```

### 💾 **Сохранение правил nftables постоянно**
```bash
# 1. Сохраните текущие правила в файл
nft list ruleset > /etc/nftables.d/custom-rules.nft

# 2. Добавьте в конфиг фаервола
echo "include \"/etc/nftables.d/custom-rules.nft\"" >> /etc/firewall.user

# 3. Примените при следующей загрузке
service firewall restart
```

## 🛡️ **3. УПРАВЛЕНИЕ СЛУЖБОЙ ФАЕРВОЛА**

```bash
# Перезапустить фаервол
service firewall restart

# Проверить статус
service firewall status

# Включить автозагрузку
/etc/init.d/firewall enable

# Выключить автозагрузку
/etc/init.d/firewall disable

# Полностью остановить фаервол (НЕ РЕКОМЕНДУЕТСЯ!)
service firewall stop

# Запустить снова
service firewall start
```

## 📋 **4. ПОЛЕЗНЫЕ ШАБЛОНЫ ПРАВИЛ**

### **Для игровых консолей/сервисов:**
```bash
# Xbox Live
uci add firewall rule
uci set firewall.@rule[-1].name='Xbox-Live'
uci set firewall.@rule[-1].src='wan'
uci set firewall.@rule[-1].proto='tcpudp'  # Оба протокола
uci set firewall.@rule[-1].dest_port='3074'
uci set firewall.@rule[-1].target='ACCEPT'

# Steam
uci add firewall rule
uci set firewall.@rule[-1].name='Steam'
uci set firewall.@rule[-1].src='wan'
uci set firewall.@rule[-1].proto='tcpudp'
uci set firewall.@rule[-1].dest_port='27000-27100'
uci set firewall.@rule[-1].target='ACCEPT'
```

### **Для веб-сервера:**
```bash
# Проброс HTTP/HTTPS
uci add firewall redirect
uci set firewall.@redirect[-1].name='Web-Server'
uci set firewall.@redirect[-1].src='wan'
uci set firewall.@redirect[-1].proto='tcp'
uci set firewall.@redirect[-1].src_dport='80 443'
uci set firewall.@redirect[-1].dest_ip='192.168.1.10'
uci set firewall.@redirect[-1].dest_port='80 443'
```

### **Запретить устройству выход в интернет:**
```bash
uci add firewall rule
uci set firewall.@rule[-1].name='Block-Device'
uci set firewall.@rule[-1].src='lan'
uci set firewall.@rule[-1].src_ip='192.168.1.50'  # IP устройства
uci set firewall.@rule[-1].dest='wan'
uci set firewall.@rule[-1].target='REJECT'  # Или DROP
```

## ⚠️ **ВАЖНЫЕ ПРЕДУПРЕЖДЕНИЯ**

1. **Всегда делайте бэкап перед изменениями:**
   ```bash
   cp /etc/config/firewall /etc/config/firewall.backup
   ```

2. **Если заблокировали себя:**
   ```bash
   # Подключитесь по кабелю и выполните:
   uci set firewall.@rule[НЕПРАВИЛЬНОЕ_ПРАВИЛО].enabled='0'
   uci commit firewall
   service firewall restart
   ```

3. **Порядок правил ВАЖЕН!** Правила обрабатываются сверху вниз. Используйте параметр `index` для указания позиции:
   ```bash
   uci -P /var/state add firewall rule  # Добавить в начало
   ```

4. **Проверяйте логи:**
   ```bash
   logread | grep firewall
   dmesg | grep nft
   ```

## 💎 **КРАТКИЙ ЧЕК-ЛИСТ ДЕЙСТВИЙ**

1. **Просмотреть** → `uci show firewall | grep rule`
2. **Добавить** → `uci add firewall rule` + настройка параметров
3. **Изменить** → `uci set firewall.@rule[ID].ПАРАМЕТР=ЗНАЧЕНИЕ`
4. **Удалить** → `uci delete firewall.@rule[ID]`
5. **Применить** → `uci commit firewall && service firewall restart`
6. **Проверить** → `nft list chain inet fw4 input`

**Главное правило:** Меняйте фаервол через `uci` — это сохранит настройки после перезагрузки. Прямые команды `nft` — только для временных тестов!

--------------------------




