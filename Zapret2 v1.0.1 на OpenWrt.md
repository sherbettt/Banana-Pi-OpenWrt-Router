# Полное руководство по установке и настройке Zapret2 на OpenWrt One

## Введение

Zapret2 — это инструмент для обхода Deep Packet Inspection (DPI), который используют провайдеры для блокировки или ограничения доступа к сайтам и сервисам. 
В этом руководстве я расскажу, как установить Zapret2 на роутер OpenWrt One (архитектура `aarch64_cortex-a53`, прошивка 25.12.4), 
как его правильно настроить и, главное, как менять стратегии фильтрации для решения проблем с конкретными сервисами.

---

## Часть 1. Установка Zapret2

### 1.1. Скачивание и распаковка

```bash
# Переходим в удобное место (например, на NVMe диск)
cd /mnt/nvme/utils

# Скачиваем последнюю версию (флаг -L нужен для следования редиректам GitHub)
wget -O zapret2.tar.gz https://github.com/bol-van/zapret2/releases/download/v1.0.1/zapret2-v1.0.1-openwrt-embedded.tar.gz

# Распаковываем
tar -xzf zapret2.tar.gz
cd zapret2-v1.0.1
```

### 1.2. Запуск установки

Запускаем главный интерактивный скрипт:

```bash
chmod +x install_easy.sh
./install_easy.sh
```

Скрипт задаст несколько вопросов. Вот оптимальные ответы для OpenWrt One:

| Вопрос | Рекомендуемый ответ | Пояснение |
|--------|---------------------|-----------|
| Копировать в `/opt/zapret2`? | `Y` | Скрипт скопирует себя в стандартное место |
| Тип фаервола | `2` (nftables) | OpenWrt 25.12 использует nftables по умолчанию |
| Поддержка IPv6 | оставить пустым (Enter) | Для большинства задач достаточно IPv4 |
| GNU sort | `N` | Работает и со стандартным busybox sort |
| Метод фильтрации | `3` (hostlist) | Самый эффективный для списка доменов |
| Включить nfqws2 | `Y` | Основной движок обхода DPI |
| Редактировать опции | `Y` (для тонкой настройки) или `N` | Позже можно будет изменить |
| Автообновление списков | `Y` | Автоматически подгружает актуальные домены |
| Выбор источника списков | `2` (get_antizapret_domains.sh) | Наиболее полный список |
| Flow offloading | `1` (donttouch) | Не мешаем работе nfqws2 |

### 1.3. Что происходит после установки

После успешной установки:
- Все файлы скопированы в `/opt/zapret2/`
- Создан init-скрипт `/etc/init.d/zapret2`
- Сервис уже запущен и добавлен в автозагрузку
- Исходную папку можно удалить: `rm -rf /mnt/nvme/utils/Zapret2`

---

## Часть 2. Конфигурация Zapret2

### 2.1. Основные файлы конфигурации

| Файл | Назначение |
|------|------------|
| `/opt/zapret2/config` | Главный конфигурационный файл |
| `/opt/zapret2/ipset/zapret-hosts-user.txt` | Ваш персональный список доменов |
| `/opt/zapret2/init.d/openwrt/custom.d/` | Пользовательские скрипты |

### 2.2. Добавление своих доменов

Редактируем файл со списком доменов:

```bash
mcedit /opt/zapret2/ipset/zapret-hosts-user.txt
# или
vi /opt/zapret2/ipset/zapret-hosts-user.txt
```

Пример содержимого:

```
youtube.com
google.com
instagram.com
facebook.com
twitter.com
discord.com
github.com
```

**Важно:** После изменения списка доменов нужно перезапустить сервис:

```bash
/etc/init.d/zapret2 restart
```

### 2.3. Проверка работы

```bash
# Статус сервиса
/etc/init.d/zapret2 status

# Проверка процесса
procs | grep nfqws2

# Просмотр правил nftables
nft list ruleset | grep zapret
```

---

## Часть 3. Методы фильтрации (стратегии обхода DPI)

Zapret2 поддерживает множество методов "десинхронизации" (desync) — способов модификации пакетов, чтобы DPI провайдера не мог их корректно проанализировать.

### 3.1. Основные типы стратегий

| Стратегия | Описание | Когда использовать |
|-----------|----------|-------------------|
| **fake** | Внедрение фейковых пакетов, сбивающих анализатор DPI. Использует предустановленные "болванки" (blobs) | Универсальный метод, хорош для начала |
| **multisplit** | Разрезание TLS Client Hello на несколько частей | Эффективен против многих DPI, особенно для HTTPS |
| **disorder** | Перестановка TCP-сегментов в неправильном порядке | Для сложных случаев, когда другие методы не работают |
| **circular** | Циклическая модификация пакетов | Специфический метод, редко используется |
| **fake + multisplit** | Комбинация двух методов | Для самых сложных случаев |

### 3.2. Расшифровка вашей текущей стратегии

Вот что сейчас у вас в конфиге (параметр `NFQWS2_OPT`):

```bash
--filter-tcp=443 --filter-l7=tls 
--payload=tls_client_hello 
--lua-desync=multisplit:pos=midsld:seqovl=5
```

Разберём каждую опцию:

| Опция | Значение |
|-------|----------|
| `--filter-tcp=443` | Применять только к TCP-трафику на порт 443 (HTTPS) |
| `--filter-l7=tls` | Фильтрация по L7-протоколу TLS (HTTPS) |
| `--payload=tls_client_hello` | Модифицировать именно TLS Client Hello |
| `--lua-desync=multisplit:pos=midsld:seqovl=5` | Использовать метод multisplit с перекрытием последовательности 5 байт |

**Что это значит:** Ваш Zapret2 обрабатывает **только HTTPS-трафик** (порт 443) и применяет метод `multisplit` к TLS-рукопожатию.

### 3.3. Как менять стратегию

Стратегия задаётся в файле `/opt/zapret2/config` в переменной `NFQWS2_OPT`. Вот несколько готовых вариантов:

#### Вариант 1. Метод "fake" (простой и эффективный)

```bash
NFQWS2_OPT="
--filter-tcp=443 --filter-l7=tls 
--payload=tls_client_hello 
--lua-desync=fake:blob=fake_default_tls:tcp_md5:tcp_seq=-10000"
```

#### Вариант 2. Комбинация fake + multisplit (более надёжный) 

```bash
NFQWS2_OPT="
--filter-tcp=443 --filter-l7=tls 
--payload=tls_client_hello 
--lua-desync=fake:blob=fake_default_tls:tcp_md5:tcp_seq=-10000 
--lua-desync=multisplit:pos=midsld:seqovl=5"
```

#### Вариант 3. Полная стратегия для YouTube и Google сервисов

```bash
NFQWS2_OPT="
--filter-tcp=443 --filter-l7=tls --hostlist=/opt/zapret2/ipset/zapret-hosts-user.txt
--payload=tls_client_hello 
--lua-desync=multisplit:pos=midsld:seqovl=5"
```

**Как применить новую стратегию:**
1. Отредактируйте `/opt/zapret2/config`
2. Найдите строку `NFQWS2_OPT` и замените её содержимое
3. Перезапустите сервис: `/etc/init.d/zapret2 restart`

---

## Часть 4. Решение проблемы: YouTube грузится частично

### 4.1. Почему это происходит

Вы описали классическую ситуацию: страница YouTube открывается, превью видео видны, но само видео не грузится. Это происходит потому, что:

1. **Страница YouTube и видео — разные домены.** YouTube использует домен `youtube.com` для интерфейса, но само видео отдаётся с доменов `googlevideo.com` .
2. **Ваш список доменов** пока содержит только `youtube.com`, а домен `googlevideo.com` в нём отсутствует.
3. **DPI блокирует именно видеопотоки**, а интерфейс оставляет доступным, чтобы усложнить диагностику.

### 4.2. Решение: добавить googlevideo.com в список

```bash
mcedit /opt/zapret2/ipset/zapret-hosts-user.txt
```

Добавьте эти строки:

```
googlevideo.com
youtubei.googleapis.com
ytimg.com
ggpht.com
googleusercontent.com
```

После сохранения:

```bash
/etc/init.d/zapret2 restart
```

### 4.3. Если не помогло — смените стратегию

Если проблема осталась, попробуйте метод `fake` вместо `multisplit`. Он часто лучше работает с видеопотоками.

Редактируем `/opt/zapret2/config`:

```bash
mcedit /opt/zapret2/config
```

Найдите и замените строку `NFQWS2_OPT`:

```bash
NFQWS2_OPT="
--filter-tcp=443 --filter-l7=tls --hostlist=/opt/zapret2/ipset/zapret-hosts-user.txt
--payload=tls_client_hello 
--lua-desync=fake:blob=fake_default_tls:tcp_md5:tcp_seq=-10000"
```

Перезапускаем:

```bash
/etc/init.d/zapret2 restart
```

### 4.4. Дополнительные меры

Если ни один из методов не помог, можно проверить несколько моментов:

1. **Проверьте, нет ли конфликта с flow offloading** :
   ```bash
   cat /etc/nftables.d/* | grep flow_offload
   ```

2. **Посмотрите логи на предмет ошибок**:
   ```bash
   tail -f /opt/zapret2/logs/nfqws2.log
   ```

3. **Проверьте, обрабатываются ли пакеты**:
   ```bash
   nft list ruleset | grep -A10 "chain zapret"
   ```

---

## Часть 5. Полезные команды для дальнейшей работы

| Команда | Назначение |
|---------|------------|
| `/etc/init.d/zapret2 restart` | Перезапустить сервис после изменения конфига |
| `/etc/init.d/zapret2 stop` | Остановить сервис |
| `/etc/init.d/zapret2 start` | Запустить сервис |
| `procs \| grep nfqws2` | Проверить, работает ли процесс |
| `nft list ruleset \| grep zapret` | Посмотреть правила в фаерволе |
| `tail -f /opt/zapret2/logs/nfqws2.log` | Смотреть логи в реальном времени |
| `blockcheckw` | Просканировать оптимальную стратегию для вашего провайдера |

---

## Заключение

Вы успешно установили Zapret2 на OpenWrt One. Теперь вы знаете:

- Как менять стратегии обхода DPI через переменную `NFQWS2_OPT`
- Какие бывают методы: `fake`, `multisplit`, `disorder`
- Почему YouTube грузится частично и как это исправить

Если какая-то стратегия перестаёт работать (провайдеры постоянно обновляют свои DPI-системы), 
просто вернитесь к настройкам и выберите другой метод из списка выше.

Удачной настройки! 🚀



