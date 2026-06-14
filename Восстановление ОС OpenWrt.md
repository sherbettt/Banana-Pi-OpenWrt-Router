# Восстановление ОС OpenWrt

## 📁 Структура бэкапа: что мы сохраняем?

В нашем случае бэкап включает:

| Тип файлов | Путь | Назначение |
|------------|------|------------|
| Конфигурация | `/etc/config/*` | Все настройки (сеть, firewall, LuCI и т.д.) |
| SSH-ключи | `/etc/dropbear/dropbear_*_host_key` | Ключи для входа по SSH |
| SSL-сертификаты | `/etc/uhttpd.*` | Сертификаты веб-интерфейса |
| Системные файлы | `/etc/group`, `/etc/passwd`, `/etc/shadow` и др. | Пользователи, пароли, профили |
| Правила nftables | `/etc/nftables.d/*.nft` | Пользовательские правила файрвола |
| Список пакетов | Сохраняется в текстовый файл | Для переустановки всех пакетов |
| Репозитории | `/etc/apk/repositories*` | Источники пакетов |

---

## 🔧 Скрипты для бэкапа и восстановления

### Исходный скрипт (smart_backup.sh)

Мы использовали скрипт `smart_backup.sh`, который был найден в сети. Он оказался написан для `opkg` — старого менеджера пакетов. Вот как выглядела его проблемная часть:

```bash
# Это работало в старых версиях OpenWrt (до 25.12)
MODIFIED_CONFIGS=$(opkg list-changed-conffiles 2>/dev/null)
if opkg list-installed > $BACKUP_DIR/installed_packages.txt; then
```

**Проблема:** В OpenWrt 25.12 и новее `opkg` заменён на `apk`, а команды изменились.

### Исправленный скрипт (smart_backup_apk.sh)

Мы адаптировали скрипт для `apk`:

```bash
# Вместо opkg list-installed теперь:
apk list --installed > $BACKUP_DIR/installed_packages.txt

# Вместо поиска ключей opkg (их нет в apk):
# OPKG_KEYS=$(find /etc/opkg/keys/ -type f 2>/dev/null) — удалено

# Добавлено сохранение репозиториев apk:
cp /etc/apk/repositories $BACKUP_DIR/repositories.backup
cp /etc/apk/repositories.d/*.list $BACKUP_DIR/
```

**Ключевые отличия:**
- `apk list --installed` выводит список пакетов в формате: `package-версия архитектура`
- Репозитории теперь хранятся в `/etc/apk/repositories` и `/etc/apk/repositories.d/`
- В `apk` нет аналога `list-changed-conffiles` — все конфиги бекапим целиком

### Скрипт восстановления (smart_restore_apk.sh)

Скрипт восстановления делает обратную работу:
1. Распаковывает архив во временную папку
2. Копирует файлы обратно в систему
3. Выводит команды для переустановки пакетов (сами пакеты в архив не включаются, так как они занимают много места и всегда доступны в репозиториях)

---

## 📋 Пошаговая инструкция по созданию бэкапа

### Шаг 1. Создайте скрипт бэкапа

```bash
cat > /root/smart_backup_apk.sh << 'EOF'
[содержимое скрипта из предыдущего сообщения]
EOF
chmod +x /root/smart_backup_apk.sh
```

### Шаг 2. Запустите бэкап

```bash
./smart_backup_apk.sh
```

**Пример вывода:**
```
=== CREATING SMART BACKUP (apk version) ===

=== All Config Files ===
  ✓ /etc/config/dhcp
  ✓ /etc/config/network
  ...

=== SSH Host Keys ===
  ✓ /etc/dropbear/dropbear_ed25519_host_key
  ...

=== SAVING PACKAGE LIST (apk) ===
✓ Package list saved: /root/backup/installed_packages.txt
  Total packages: 351

=== BACKUP COMPLETED ===
Backup file: /root/backup/openwrt_backup_20260614_115352.tar.gz
Total files backed up: 30
Backup size: 8.0K
```

### Шаг 3. Сохраните бэкап на внешний носитель

```bash
# Скопируйте бэкап в папку downloads (на NVMe)
cp /root/backup/*.tar.gz /mnt/nvme/downloads/

# На компьютере (внешний бэкап)
scp -O root@192.168.7.1:/mnt/nvme/downloads/openwrt_backup_*.tar.gz ~/Downloads/
```

### Шаг 4. Что получилось в итоге

В папке `/root/backup/` появились файлы:

| Файл | Содержимое |
|------|------------|
| `openwrt_backup_YYYYMMDD_HHMMSS.tar.gz` | Архив с конфигами (30 файлов, ~8 КБ) |
| `installed_packages.txt` | Список всех 351 установленных пакетов |
| `repositories.backup` | Файл с репозиториями `/etc/apk/repositories` |
| `distfeeds.list`, `customfeeds.list` | Дополнительные файлы репозиториев |

---

## 🔄 Восстановление из бэкапа

### Шаг 1. Подготовьте скрипт восстановления

```bash
cat > /root/smart_restore_apk.sh << 'EOF'
[содержимое скрипта восстановления]
EOF
chmod +x /root/smart_restore_apk.sh
```

### Шаг 2. Запустите восстановление

```bash
./smart_restore_apk.sh /root/backup/openwrt_backup_20260614_115352.tar.gz
```

### Шаг 3. Переустановите пакеты

Скрипт выведет команды для переустановки всех пакетов:

```bash
while read pkg; do
    apk add $(echo $pkg | cut -d' ' -f1)
done < /root/backup/installed_packages.txt
```

### Шаг 4. Перезагрузите роутер

```bash
reboot
```

---

## 🧠 Что мы узнали о командах

### Сравнение opkg vs apk для бэкапа

| Действие | opkg (старый) | apk (новый) |
|----------|---------------|-------------|
| Список пакетов | `opkg list-installed` | `apk list --installed` |
| Список изменённых конфигов | `opkg list-changed-conffiles` | ❌ нет аналога (бекапим все конфиги) |
| Где лежат ключи | `/etc/opkg/keys/` | ❌ apk не использует ключи |
| Где лежат репозитории | `/etc/opkg/distfeeds.conf` | `/etc/apk/repositories` и `.d/` |

### Полезные команды для проверки

```bash
# Просмотр содержимого архива без распаковки
tar -tzf backup.tar.gz | head -20

# Проверка целостности архива
gunzip -t backup.tar.gz && echo "OK" || echo "CORRUPTED"

# Размер архива и количество файлов
echo "Size: $(du -h backup.tar.gz | cut -f1)"
echo "Files: $(tar -tzf backup.tar.gz | wc -l)"
```

---

## ⚠️ Важные замечания

1. **Размер бэкапа мал** (8-30 КБ) — это нормально, так как сохраняются только конфиги и список пакетов. Сами пакеты НЕ сохраняются, они переустанавливаются из репозиториев.

2. **Полный бэкап overlay** (если вам действительно нужно сохранить бинарники) делается так:
   ```bash
   tar -czf /mnt/nvme/overlay_backup.tar.gz /overlay/
   ```
   Но это займёт сотни мегабайт и обычно не требуется.

3. **Перед восстановлением** рекомендуется сделать резервную копию текущего состояния — мало ли что пойдёт не так.

4. **Восстановление на другой роутер** возможно, если архитектура процессора совпадает (в нашем случае `aarch64_cortex-a53`). Но конфиги могут быть несовместимы из-за разных сетевых интерфейсов.

---

## 📦 Готовые файлы

После всех манипуляций у вас в `/mnt/nvme/downloads/` лежат:

- `smart_backup_apk.sh` — скрипт для создания бэкапа
- `smart_restore_apk.sh` — скрипт для восстановления
- `openwrt_backup_YYYYMMDD_HHMMSS.tar.gz` — свежий бэкап конфигурации

---

## 🎯 Итог

Теперь вы можете:
1. Создавать бэкап состояния роутера одной командой
2. Восстанавливать конфигурацию после сброса или перепрошивки
3. Переносить настройки на другой роутер (с осторожностью)

В случае любых проблем с `apk` или потерей настроек — у вас есть точка возврата. Бэкапы — это страховка, которая окупается в первый же серьёзный сбой.

<br>

<details>
<summary>❗ Backup scripts ❗</summary>

```bash
root@OpenWrt(0):/mnt/nvme/downloads# cat smart_backup.sh
#!/bin/sh

# Директория для бэкапа
BACKUP_DIR="/root/backup"
mkdir -p $BACKUP_DIR
BACKUP_FILE="openwrt_backup_$(date +%Y%m%d_%H%M%S).tar.gz"

echo "=== CREATING SMART BACKUP ==="
echo ""

# Функция для добавления файлов в список бэкапа
add_files() {
    local description="$1"
    local files="$2"
    local count=0
    
    if [ -n "$files" ]; then
        echo "=== $description ==="
        for file in $files; do
            if [ -f "$file" ]; then
                FILES_TO_BACKUP="$FILES_TO_BACKUP $file"
                echo "  ✓ $file"
                count=$((count + 1))
            fi
        done
        [ $count -eq 0 ] && echo "  (none found)"
        echo ""
    fi
}

# Инициализация списка файлов
FILES_TO_BACKUP=""

# 1. Измененные конфиг-файлы через opkg
MODIFIED_CONFIGS=$(opkg list-changed-conffiles 2>/dev/null)
add_files "Modified Config Files" "$MODIFIED_CONFIGS"

# 2. Все файлы в /etc/config/
ALL_CONFIGS=$(find /etc/config/ -type f 2>/dev/null)
add_files "All Config Files" "$ALL_CONFIGS"

# 3. SSH ключи
SSH_KEYS=$(find /etc/dropbear/ -name "dropbear_*_host_key" -type f 2>/dev/null)
add_files "SSH Host Keys" "$SSH_KEYS"

# 4. SSL сертификаты uhttpd
UHTTPD_CERTS=$(find /etc/ -name "uhttpd.*" -type f 2>/dev/null)
add_files "uHTTPd Certificates" "$UHTTPD_CERTS"

# 5. Ключи opkg
OPKG_KEYS=$(find /etc/opkg/keys/ -type f 2>/dev/null)
add_files "OPKG Keys" "$OPKG_KEYS"

# 6. Пользовательские crontabs
CRONTABS=$(find /etc/crontabs/ -type f 2>/dev/null)
add_files "Crontabs" "$CRONTABS"

# 7. Конфиги sing-box
SING_BOX_CONFIGS=$(find /etc/sing-box/ -name "*.json" -type f 2>/dev/null)
add_files "Sing-box Configs" "$SING_BOX_CONFIGS"

# 8. Важные системные файлы
SYSTEM_FILES="
/etc/group
/etc/passwd
/etc/shadow
/etc/hosts
/etc/shells
/etc/profile
/etc/rc.local
/etc/sysctl.conf
/etc/inittab
/etc/shinit
"
add_files "System Files" "$SYSTEM_FILES"

# 9. Пользовательские nftables правила
NFTABLES_RULES=$(find /etc/nftables.d/ -name "*.nft" -type f 2>/dev/null)
add_files "NFTables Rules" "$NFTABLES_RULES"

# Создаем архив
echo "=== CREATING ARCHIVE ==="
echo "File: $BACKUP_DIR/$BACKUP_FILE"

if tar -czf $BACKUP_DIR/$BACKUP_FILE $FILES_TO_BACKUP 2>/dev/null; then
    echo "✓ Archive created successfully"
else
    echo "✗ Failed to create archive"
    exit 1
fi

# Создаем список установленных пакетов
echo ""
echo "=== SAVING PACKAGE LIST ==="
if opkg list-installed > $BACKUP_DIR/installed_packages.txt; then
    echo "✓ Package list saved: $BACKUP_DIR/installed_packages.txt"
    echo "  Total packages: $(wc -l < $BACKUP_DIR/installed_packages.txt)"
else
    echo "✗ Failed to save package list"
fi

# Итоговая информация
echo ""
echo "=== BACKUP COMPLETED ==="
echo "Backup file: $BACKUP_DIR/$BACKUP_FILE"
echo "Total files backed up: $(echo $FILES_TO_BACKUP | wc -w)"
echo "Backup size: $(du -h $BACKUP_DIR/$BACKUP_FILE | cut -f1)"
echo ""
echo "To restore, use: smart_restore.sh $BACKUP_FILE"
```

```bash
root@OpenWrt(0):/mnt/nvme/downloads# cat smart_backup_apk.sh 
#!/bin/sh

# Директория для бэкапа
BACKUP_DIR="/root/backup"
mkdir -p $BACKUP_DIR
BACKUP_FILE="openwrt_backup_$(date +%Y%m%d_%H%M%S).tar.gz"

echo "=== CREATING SMART BACKUP (apk version) ==="
echo ""

# Функция для добавления файлов в список бэкапа
add_files() {
    local description="$1"
    local files="$2"
    local count=0
    
    if [ -n "$files" ]; then
        echo "=== $description ==="
        for file in $files; do
            if [ -f "$file" ]; then
                FILES_TO_BACKUP="$FILES_TO_BACKUP $file"
                echo "  ✓ $file"
                count=$((count + 1))
            fi
        done
        [ $count -eq 0 ] && echo "  (none found)"
        echo ""
    fi
}

# Инициализация списка файлов
FILES_TO_BACKUP=""

# 1. Все файлы в /etc/config/
ALL_CONFIGS=$(find /etc/config/ -type f 2>/dev/null)
add_files "All Config Files" "$ALL_CONFIGS"

# 2. SSH ключи
SSH_KEYS=$(find /etc/dropbear/ -name "dropbear_*_host_key" -type f 2>/dev/null)
add_files "SSH Host Keys" "$SSH_KEYS"

# 3. SSL сертификаты uhttpd
UHTTPD_CERTS=$(find /etc/ -name "uhttpd.*" -type f 2>/dev/null)
add_files "uHTTPd Certificates" "$UHTTPD_CERTS"

# 4. Пользовательские crontabs
CRONTABS=$(find /etc/crontabs/ -type f 2>/dev/null)
add_files "Crontabs" "$CRONTABS"

# 5. Важные системные файлы
SYSTEM_FILES="
/etc/group
/etc/passwd
/etc/shadow
/etc/hosts
/etc/shells
/etc/profile
/etc/rc.local
/etc/sysctl.conf
/etc/inittab
/etc/shinit
"
add_files "System Files" "$SYSTEM_FILES"

# 6. Пользовательские nftables правила
NFTABLES_RULES=$(find /etc/nftables.d/ -name "*.nft" -type f 2>/dev/null)
add_files "NFTables Rules" "$NFTABLES_RULES"

# 7. Дополнительные конфиги (если есть)
if [ -f /etc/uclient-fetch.conf ]; then
    add_files "uclient-fetch Config" "/etc/uclient-fetch.conf"
fi

# Создаем архив
echo "=== CREATING ARCHIVE ==="
echo "File: $BACKUP_DIR/$BACKUP_FILE"

if tar -czf $BACKUP_DIR/$BACKUP_FILE $FILES_TO_BACKUP 2>/dev/null; then
    echo "✓ Archive created successfully"
else
    echo "✗ Failed to create archive"
    exit 1
fi

# Создаем список установленных пакетов через apk
echo ""
echo "=== SAVING PACKAGE LIST (apk) ==="
apk list --installed > $BACKUP_DIR/installed_packages.txt 2>/dev/null
if [ -s $BACKUP_DIR/installed_packages.txt ]; then
    echo "✓ Package list saved: $BACKUP_DIR/installed_packages.txt"
    echo "  Total packages: $(wc -l < $BACKUP_DIR/installed_packages.txt)"
else
    echo "✗ Failed to save package list"
fi

# Сохраняем список репозиториев
echo ""
echo "=== SAVING REPOSITORY LIST ==="
cp /etc/apk/repositories $BACKUP_DIR/repositories.backup 2>/dev/null
cp /etc/apk/repositories.d/*.list $BACKUP_DIR/ 2>/dev/null
echo "✓ Repository configs saved"

# Итоговая информация
echo ""
echo "=== BACKUP COMPLETED ==="
echo "Backup file: $BACKUP_DIR/$BACKUP_FILE"
echo "Total files backed up: $(echo $FILES_TO_BACKUP | wc -w)"
echo "Backup size: $(du -h $BACKUP_DIR/$BACKUP_FILE | cut -f1)"
echo ""
echo "To restore, use: smart_restore_apk.sh $BACKUP_FILE"
```

```bash
root@OpenWrt(0):/mnt/nvme/downloads# cat smart_restore_apk.sh 
#!/bin/sh

RESTORE_FILE="$1"

if [ -z "$RESTORE_FILE" ]; then
    echo "Usage: $0 backup_file.tar.gz"
    echo ""
    echo "Available backups:"
    ls -la /root/backup/*.tar.gz 2>/dev/null
    exit 1
fi

if [ ! -f "$RESTORE_FILE" ]; then
    echo "Error: File $RESTORE_FILE not found"
    exit 1
fi

echo "=== RESTORING FROM BACKUP ==="
echo "Restore file: $RESTORE_FILE"

TEMP_DIR="/tmp/restore_$$"
mkdir -p $TEMP_DIR

# Извлекаем архив
tar -xzf "$RESTORE_FILE" -C $TEMP_DIR

# Восстанавливаем конфиги
if [ -d "$TEMP_DIR/etc/config" ]; then
    cp -f $TEMP_DIR/etc/config/* /etc/config/ 2>/dev/null
    echo "✓ Restored config files"
fi

# Восстанавливаем SSH ключи
if [ -d "$TEMP_DIR/etc/dropbear" ]; then
    cp -f $TEMP_DIR/etc/dropbear/* /etc/dropbear/ 2>/dev/null
    echo "✓ Restored dropbear keys"
fi

# Восстанавливаем сертификаты
for cert in uhttpd.crt uhttpd.key; do
    if [ -f "$TEMP_DIR/etc/$cert" ]; then
        cp -f "$TEMP_DIR/etc/$cert" "/etc/$cert"
        echo "✓ Restored $cert"
    fi
done

# Восстанавливаем системные файлы
for file in group passwd shadow hosts shells profile rc.local sysctl.conf inittab shinit; do
    if [ -f "$TEMP_DIR/etc/$file" ]; then
        cp -f "$TEMP_DIR/etc/$file" "/etc/$file"
        echo "✓ Restored $file"
    fi
done

# Восстанавливаем nftables правила
if [ -d "$TEMP_DIR/etc/nftables.d" ]; then
    mkdir -p /etc/nftables.d
    cp -f $TEMP_DIR/etc/nftables.d/*.nft /etc/nftables.d/ 2>/dev/null
    echo "✓ Restored nftables rules"
fi

# Показываем список пакетов для переустановки
if [ -f "$TEMP_DIR/installed_packages.txt" ]; then
    echo ""
    echo "=== PACKAGES TO REINSTALL ==="
    echo "To reinstall all packages, run:"
    echo ""
    echo "  while read pkg; do"
    echo "    apk add \$(echo \$pkg | cut -d' ' -f1)"
    echo "  done < $TEMP_DIR/installed_packages.txt"
    echo ""
fi

# Очистка
rm -rf $TEMP_DIR

echo ""
echo "=== RESTORE COMPLETED ==="
echo "Please reboot your router for changes to take effect"
```

</details> 
<br/>
