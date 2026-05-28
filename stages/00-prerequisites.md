# 🐧 Этап 00. Prerequisites — Linux, процессы, сети

> **Цель:** заложить фундамент. Контейнеры — это процессы Linux. Без понимания Linux ты не поймёшь Docker.

---

## ⏱️ Длительность

3-5 дней (по 2-3 часа в день)

## 📋 Чеклист готовности

- [ ] Знаю что такое процесс, PID, родитель, демон
- [ ] Умею работать с stdin/stdout/stderr и pipe
- [ ] Понимаю права доступа (chmod, chown, umask)
- [ ] Знаю как работают namespaces (минимум: mount, net, pid)
- [ ] Знаю что такое cgroups
- [ ] Умею читать iptables / nftables
- [ ] Понимаю как работает DNS и /etc/hosts
- [ ] Знаю как смотреть открытые порты (ss, lsof)

---

## 📚 Содержание

1. [Процессы и сигналы](#1-процессы-и-сигналы)
2. [Файловая система и mount](#2-файловая-система-и-mount)
3. [Права доступа](#3-права-доступа)
4. [Namespaces — основа контейнеров](#4-namespaces--основа-контейнеров)
5. [Cgroups — лимиты ресурсов](#5-cgroups--лимиты-ресурсов)
6. [Сети Linux](#6-сети-linux)
7. [Capabilities](#7-capabilities)
8. [Практика](#8-практика)

---

## 1. Процессы и сигналы

Контейнер = процесс с особым окружением. PID 1 в контейнере — это твой главный процесс.

```bash
# Дерево процессов
ps -ef --forest
pstree -p

# Сигналы
kill -SIGTERM <pid>   # вежливая остановка (так Docker останавливает контейнер)
kill -SIGKILL <pid>   # принудительная (так Docker делает после 10s grace period)

# Что произойдёт, если PID 1 не обрабатывает SIGTERM?
# → Docker подождёт 10 секунд → SIGKILL → данные могут потеряться
```

**Важно:** в контейнере твоё приложение — это PID 1. Оно должно обрабатывать SIGTERM, иначе `docker stop` будет жёстко убивать процесс.

## 2. Файловая система и mount

Docker использует overlayfs для слоёв образа.

```bash
# Посмотреть mount-ы
mount | head
findmnt

# Виды mount-ов в контейнерах:
# - bind mount: проброс директории хоста
# - volume: managed Docker volume
# - tmpfs: in-memory
# - overlay: слои образа
```

## 3. Права доступа

```bash
# Octal: rwxrwxrwx → 755
chmod 644 file
chown user:group file

# Особое: SUID/SGID/sticky
ls -l /usr/bin/sudo   # -rwsr-xr-x (SUID)
```

В контейнерах UID хоста и UID контейнера — одно и то же ядро. Поэтому root в контейнере = root на хосте, если не используешь user namespaces.

## 4. Namespaces — основа контейнеров

Namespaces изолируют ресурсы. Контейнер — это процесс в своих namespaces.

| Namespace | Что изолирует |
|-----------|---------------|
| mnt | mount points (своя файловая система) |
| pid | PID-пространство (свой PID 1) |
| net | сетевые интерфейсы, маршруты |
| ipc | shared memory, semaphores |
| uts | hostname, domainname |
| user | UID/GID mapping |
| cgroup | cgroup-иерархия |
| time | системное время (новый) |

```bash
# Создать namespace вручную
sudo unshare --net --pid --fork --mount-proc bash
# теперь ты в новом сетевом и PID namespace

# Посмотреть namespaces процесса
ls -la /proc/$$/ns/
```

## 5. Cgroups — лимиты ресурсов

Cgroups (control groups) ограничивают CPU, RAM, IO для группы процессов.

```bash
# cgroups v2 (актуально для 2026)
ls /sys/fs/cgroup/

# Лимит памяти на контейнер
docker run --memory=512m --cpus=1.5 nginx
# → создаётся cgroup с этими лимитами
```

**Что произойдёт при OOM:** ядро вызовет OOM-killer внутри cgroup. Без лимитов один контейнер может съесть всю память хоста.

## 6. Сети Linux

```bash
# Интерфейсы
ip a
ip link

# Маршруты
ip route

# Сокеты
ss -tlnp           # TCP listening + процессы
ss -ulnp           # UDP

# DNS
cat /etc/resolv.conf
dig example.com
```

Docker создаёт bridge `docker0` и veth-пары для каждого контейнера. Понимание bridge, veth, NAT и iptables даст тебе свободу в дебаге сетевых проблем.

## 7. Capabilities

Linux capabilities — разделение прав root на 40+ маленьких прав.

```bash
# Посмотреть capabilities процесса
getpcaps $$

# Docker по умолчанию даёт ограниченный набор
docker run --rm alpine sh -c 'cat /proc/1/status | grep Cap'

# Минимизируй: убирай всё, добавляй только нужное
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE nginx
```

---

## 8. Практика

### Упражнение 1: процессы и сигналы
Напиши bash-скрипт, который ловит SIGTERM и логирует "graceful shutdown" перед выходом. Запусти и пошли `kill -TERM`.

### Упражнение 2: namespaces
Через `unshare` создай новый net namespace. Проверь, что внутри нет интерфейсов кроме lo. Подними lo (`ip link set lo up`).

### Упражнение 3: cgroups
Через systemd-run или прямую запись в `/sys/fs/cgroup/` создай группу с лимитом 100MB и запусти в ней stress-ng. Получи OOM.

### Упражнение 4: сеть
Создай два net namespace, соедини их через veth-пару, настрой IP — пингани из одного в другой.

### Упражнение 5: capabilities
Запусти контейнер с `--cap-drop=ALL` и попробуй `ping`. Понадобится `--cap-add=NET_RAW`. Запусти tcpdump — поймёшь зачем NET_ADMIN.

---

## 📚 Ресурсы

### Telegram
- 🔥 [ai_machinelearning_big_data](https://t.me/ai_machinelearning_big_data)
- 🐍 [pythonl](https://t.me/pythonl)
- 📚 [Папка каналов](https://t.me/addlist/8vDUwYRGujRmZjFi)

### Книги (бесплатно)
- [The Linux Command Line](http://linuxcommand.org/tlcl.php) — William Shotts
- [Linux Inside](https://github.com/0xAX/linux-insides) — устройство ядра, бесплатно на GitHub

### Видео
- [Liz Rice — Containers From Scratch](https://www.youtube.com/watch?v=8fi7uSYlOdc) — собрать контейнер без Docker
- [Jérôme Petazzoni — Container Networking](https://www.youtube.com/watch?v=6v_BDHIgOY8)

### Документация
- [man-pages namespaces(7)](https://man7.org/linux/man-pages/man7/namespaces.7.html)
- [man-pages cgroups(7)](https://man7.org/linux/man-pages/man7/cgroups.7.html)
- [man-pages capabilities(7)](https://man7.org/linux/man-pages/man7/capabilities.7.html)

### Tools
- [bcc / bpftrace](https://github.com/iovisor/bpftrace) — eBPF tracing
- [ctop](https://github.com/bcicen/ctop) — top для контейнеров
- [lazydocker](https://github.com/jesseduffield/lazydocker)

---

## ➡️ Что дальше

[Этап 01 — Fundamentals](01-fundamentals.md): что такое контейнер и OCI.

[⬅ MAP](../MAP.md) · [⬅ README](../README.md)
