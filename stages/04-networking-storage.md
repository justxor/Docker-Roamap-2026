# 🌐 Этап 04. Networking & Storage

> **Цель:** глубокое понимание сетей контейнеров (bridge, host, overlay, macvlan, ipvlan) и хранения (volumes, bind, tmpfs, drivers).

---

## ⏱️ Длительность

4-6 дней

## 📋 Чеклист готовности

- [ ] Знаю драйверы сетей и когда какой использовать
- [ ] Понимаю как работает bridge и NAT под капотом
- [ ] Умею дебажить сетевые проблемы (tcpdump, nsenter, traceroute)
- [ ] Знаю разницу volume vs bind vs tmpfs
- [ ] Умею бэкапить и восстанавливать volumes
- [ ] Знаю про volume drivers (local, nfs, cifs, plugins)

---

## 1. Network drivers

### bridge (по умолчанию)
```bash
docker network ls
docker network inspect bridge
ip a show docker0    # видишь bridge-интерфейс с IP 172.17.0.1
```

Каждый контейнер получает veth-пару: один конец в namespace контейнера (eth0), другой — на хосте, подключен к bridge.

### host
```bash
docker run --network host nginx
```
Контейнер делит net-namespace с хостом. Без NAT, без изоляции, максимальная производительность. **Не работает на Docker Desktop macOS/Windows** (там VM).

### none
```bash
docker run --network none alpine ip a
# только lo, никакой сети
```

### overlay (Swarm / multi-host)
Сети поверх нескольких хостов. Используется в Docker Swarm. В k8s — другие решения (CNI).

### macvlan / ipvlan
Контейнер получает MAC-адрес в сети хоста, выглядит как отдельная машина в LAN.

## 2. User-defined networks

```bash
docker network create --driver bridge mynet
docker run -d --name web --network mynet nginx
docker run -it --rm --network mynet alpine ping web
# в user-defined сети работает DNS по именам сервисов
```

В `docker compose` это происходит автоматически.

## 3. Port publishing

```bash
docker run -p 8080:80 nginx           # 8080 на хосте → 80 в контейнере (TCP)
docker run -p 127.0.0.1:8080:80 nginx # только на localhost
docker run -p 8080:80/udp nginx       # UDP
docker run -P nginx                   # автоматически на случайные порты
```

Под капотом — iptables NAT-правила:
```bash
sudo iptables -t nat -L DOCKER -n
```

## 4. DNS в контейнере

```bash
docker run --rm alpine cat /etc/resolv.conf
# nameserver 127.0.0.11 — Docker's embedded DNS
# который пересылает в DNS хоста
```

В user-defined сети DNS резолвит имена контейнеров.

## 5. Сетевая отладка

```bash
# Войти в net namespace контейнера и смотреть как изнутри
docker exec -it mycontainer sh
ip a
ss -tlnp
curl -v http://other-service:8000

# С хоста — войти в namespace контейнера через nsenter
sudo nsenter -t \$(docker inspect -f '{{.State.Pid}}' mycontainer) -n ip a

# Снять трафик контейнера
sudo tcpdump -i any -nn host \$(docker inspect -f '{{.NetworkSettings.IPAddress}}' mycontainer)

# Или через docker run с capabilities
docker run --rm --net container:mycontainer nicolaka/netshoot tcpdump -i any
```

[netshoot](https://github.com/nicolaka/netshoot) — must-have образ для дебага сетей.

## 6. Storage: 3 типа

### bind mount
```bash
docker run -v /host/path:/container/path nginx
```
Проброс директории хоста. Удобно для dev. Привязка к пути хоста = непортируемо.

### named volume
```bash
docker volume create mydata
docker run -v mydata:/var/lib/postgresql/data postgres
```
Managed Docker. Лежит в `/var/lib/docker/volumes/`. Портируется через `docker volume` команды.

### tmpfs
```bash
docker run --tmpfs /tmp:size=100m nginx
```
В памяти. Быстро, эфемерно. Хорошо для секретов в runtime.

## 7. Volume drivers

```bash
# Локальный (по умолчанию)
docker volume create --driver local mydata

# NFS
docker volume create --driver local \
  --opt type=nfs \
  --opt o=addr=10.0.0.1,rw \
  --opt device=:/exports/data mydata

# Плагины: rexray, portworx, longhorn (в k8s)
```

## 8. Бэкап / восстановление volume

```bash
# Backup
docker run --rm \
  -v mydata:/source:ro \
  -v \$(pwd):/backup \
  alpine tar czf /backup/mydata.tar.gz -C /source .

# Restore
docker run --rm \
  -v mydata:/target \
  -v \$(pwd):/backup \
  alpine sh -c 'cd /target && tar xzf /backup/mydata.tar.gz'
```

## 9. Performance tips

- На macOS bind mounts медленные → используй named volumes для node_modules, .venv
- Используй `:cached` или `:delegated` на macOS (legacy, но всё ещё помогает)
- В Linux bind = почти zero-cost
- VirtioFS на Docker Desktop macOS/Windows ускоряет bind в 2-3 раза

## 10. Capabilities + security для сети

```bash
# Запретить --net host
# - запускать только с --network user-defined
# - использовать NetworkPolicy в k8s

# Доступ к raw сокетам (для ping, tcpdump)
docker run --cap-add=NET_RAW alpine ping google.com

# Полный сетевой админ-доступ
docker run --cap-add=NET_ADMIN alpine ip link add veth1 type veth
```

---

## 🧪 Практика

### Упражнение 1: bridge внутрь
Запусти 2 контейнера в одной user-defined сети. Войди в один — пингани второй по имени. Сделай tcpdump на bridge с хоста.

### Упражнение 2: namespace dive
Запусти контейнер. Найди его PID. Через nsenter войди в net namespace, посмотри `ip a`, `ip route`, `iptables -L`.

### Упражнение 3: NAT
Опубликуй порт 8080 на хосте. Найди соответствующее правило в `iptables -t nat`.

### Упражнение 4: backup volume
Подними postgres с volume. Сделай dump БД. Сделай backup volume. Восстанови на другой машине.

### Упражнение 5: netshoot
`docker run --rm -it --net container:mycontainer nicolaka/netshoot` — изучи команды (tcpdump, dig, curl, ss, mtr).

---

## 📚 Ресурсы

### Telegram
- 🔥 [ai_machinelearning_big_data](https://t.me/ai_machinelearning_big_data)
- 🐍 [pythonl](https://t.me/pythonl)
- 📚 [Папка каналов](https://t.me/addlist/8vDUwYRGujRmZjFi)

### Документация
- [Docker Networking](https://docs.docker.com/network/)
- [Docker Storage](https://docs.docker.com/storage/)

### Tools
- [netshoot](https://github.com/nicolaka/netshoot)
- [docker-volume-backup](https://github.com/offen/docker-volume-backup)

### Видео
- [Jérôme Petazzoni — Container Networking Deep Dive](https://www.youtube.com/watch?v=6v_BDHIgOY8)

---

## ➡️ Что дальше

[Этап 05 — Security](05-security.md): supply chain, SBOM, signing, scanning.

[⬅ MAP](../MAP.md) · [⬅ README](../README.md)
