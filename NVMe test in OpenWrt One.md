## 1. **Проверка состояния диска**:
```
opkg update
opkg install smartmontools
```
После установки:
```
smartctl -a /dev/nvme0n1
```

## 2. **Бенчмарк диска**:
```
opkg install iperf3 hdparm
```
Тесты:
```
# Тест последовательной скорости чтения
hdparm -Tt /dev/nvme0n1

# Тест через dd
dd if=/dev/zero of=/mnt/nvme/testfile bs=1M count=1024 conv=fdatasync
```

## 3. **Мониторинг диска**:
```
opkg install iostat
```
Запуск мониторинга:
```
iostat -x 1
```

## 4. **Файловые операции**:
```
opkg install rsync coreutils-stat
```

## 5. **Если планируете использовать для хранения**:
- Для кэширования DNS/пакетов:
```
opkg install dnsmasq-full
```
- Для хранения логов:
```
opkg install logd
```
- Для торрентов или загрузок:
```
opkg install transmission-daemon
```

## 6. **Дополнительные утилиты**:
```
opkg install fio          # расширенное тестирование производительности
opkg install e2fsprogs    # утилиты ext4 (если будете форматировать)
opkg install ntfs-3g      # для поддержки NTFS
```

## Пример сценариев тестирования:

**Тест производительности с fio**:
```bash
opkg install fio
fio --name=test --filename=/mnt/nvme/test --size=1G --readwrite=randread --bs=4k --iodepth=64 --runtime=30
```

**Тест записи**:
```bash
time dd if=/dev/zero of=/mnt/nvme/test.img bs=1G count=1 oflag=dsync
```

## Важные моменты:
1. Проверьте температуру диска:
```bash
opkg install lm-sensors
sensors
```

2. Убедитесь, что диск монтируется при загрузке (проверьте `/etc/config/fstab`).

3. Если планируете использовать под swap:
```bash
dd if=/dev/zero of=/mnt/nvme/swapfile bs=1M count=4096
mkswap /mnt/nvme/swapfile
swapon /mnt/nvme/swapfile
```



