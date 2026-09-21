# Развертывание Home Assistant через VPS с внешним HTTPS-доступом

Полное руководство по настройке удаленного защищенного доступа к Home Assistant OS (HAOS) через VPS с использованием Caddy (авто-SSL), OpenVPN и отказоустойчивого SSH-туннеля.

---

## 📋 Исходные параметры (пример)

| Параметр | Значение в примере | Описание |
| :--- | :--- | :--- |
| **VPS IP** | `95.100.200.50` | Внешний статический IP вашего VPS |
| **Домен** | `dacha.djezzy.ru` | Домен/поддомен для доступа к HA |
| **HA Local IP** | `192.168.2.113` | IP-адрес Home Assistant в домашней сети |
| **HA Local Port** | `80` | Локальный веб-порт Home Assistant |
| **Tunnel Port** | `8080` | Локальный порт на VPS для проксирования |

---

## ЧАСТЬ 1. Настройка нового VPS

### 1.1. Обновление системы
Подключитесь к VPS по SSH:
```bash
ssh root@95.100.200.50
apt update && apt upgrade -y
```

### 1.2. Установка OpenVPN сервера
Запустите проверенный скрипт автоматической установки:
```bash
wget https://git.io/vpn -O openvpn-install.sh
chmod +x openvpn-install.sh
./openvpn-install.sh
```

**Параметры мастера установки:**
- `IP address`: нажмите **Enter** (выберется внешний IP)
- `Public IPv4 or hostname`: нажмите **Enter**
- `Protocol`: выберите **1** (UDP)
- `Port`: **1194** (Enter)
- `DNS`: выберите **1** или **3** (Google)
- `Client name`: укажите `dacha-client`
- `Passwordless client`: выберите **1** (без пароля)

Скрипт сгенерирует файл `/root/dacha-client.ovpn`.

### 1.3. Включение маршрутизации ядра (IP Forwarding)
```bash
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p
```

### 1.4. Установка Caddy
```bash
apt install -y debian-keyring debian-archive-keyring apt-transport-https curl
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | tee /etc/apt/sources.list.d/caddy-stable.list
apt update
apt install -y caddy
systemctl enable caddy
```

### 1.5. Конфигурация Caddy
Откройте файл конфигурации:
```bash
nano /etc/caddy/Caddyfile
```

Полностью очистите файл и запишите следующий блок (удаление заголовков предотвращает ошибку `400: Bad Request` в HA):
```caddy
dacha.djezzy.ru {
    reverse_proxy localhost:8080 {
        header_up -X-Forwarded-For
        header_up -X-Forwarded-Proto
        header_up -X-Forwarded-Host
        header_up -X-Real-IP
    }
}
```
Сохраните изменения (`Ctrl+O`, `Enter`, `Ctrl+X`) и перезапустите службу:
```bash
systemctl restart caddy
```

### 1.6. Настройка SSH для мгновенного сброса зависших туннелей
```bash
nano /etc/ssh/sshd_config
```
Добавьте в конец файла:
```ini
ClientAliveInterval 10
ClientAliveCountMax 3
```
Сохраните файл и перезапустите SSH-демон:
```bash
systemctl restart ssh
```

### 1.7. Настройка брандмауэра (если включен UFW)
```bash
ufw allow 22/tcp
ufw allow 80/tcp
ufw allow 443/tcp
ufw allow 1194/udp
ufw reload
```

---

## ЧАСТЬ 2. Настройка DNS

В панели регистратора/DNS-провайдера вашего домена добавьте запись:
- **Тип:** `A`
- **Имя (Subdomain):** `dacha` (для `dacha.djezzy.ru`)
- **Значение (IP):** `95.100.200.50`
- **TTL:** Минимальный (например, 300 сек)

Проверьте с домашнего ПК:
```bash
ping dacha.djezzy.ru
```

---

## ЧАСТЬ 3. Настройка Home Assistant (HAOS)

### 3.1. Установка необходимых дополнений (Add-ons)
В меню **Настройки** -> **Дополнения** установите:
1. **File Editor** (из официального магазина).
2. **Advanced SSH & Web Terminal** (из официального магазина).
3. **OpenVPN Client** (добавьте сторонний репозиторий: `https://github.com/rofra/homeassistant-openvpn-client-addon`).

У всех трех дополнений включите тумблеры:
- `Запуск при загрузке` (Start on boot)
- `Ватчдог` (Watchdog)

### 3.2. Загрузка и модификация профиля OpenVPN
1. Скачайте файл `/root/dacha-client.ovpn` с VPS на свой ПК:
   ```bash
   scp root@95.100.200.50:/root/dacha-client.ovpn .
   ```
2. Через **File Editor** в папке `/config/` создайте каталог `openvpn` (путь `/config/openvpn/`).
3. Загрузите туда файл `dacha-client.ovpn`.
4. Откройте файл в **File Editor** и **в самый конец** допишите:
   ```ini
   pull-filter ignore "redirect-gateway"
   pull-filter ignore "route 192.168.2.0 255.255.255.0"
   ```
   *(Это предотвращает перехват VPN-сервером вашей локальной подсети и сохраняет доступность HA в LAN).*

### 3.3. Запуск OpenVPN Client Add-on
В настройках дополнения **OpenVPN Client** (вкладка «Конфигурация») укажите:
- `Path to OpenVPN config`: `/config/openvpn/dacha-client.ovpn`
- Остальные поля оставьте пустыми.

Нажмите **Сохранить** и **Запустить**. В журнале должно быть:
```text
Initialization Sequence Completed
```

### 3.4. Генерация ключа авторизации SSH
Откройте веб-терминал **Advanced SSH & Web Terminal** и выполните:
```bash
ssh-keygen -t ed25519 -f /config/openvpn/id_dacha -N ""
cat /config/openvpn/id_dacha.pub
```
Скопируйте выведенную строку (начинается с `ssh-ed25519 ...`).

### 3.5. Добавление открытого ключа на VPS
На VPS откройте:
```bash
nano ~/.ssh/authorized_keys
```
Вставьте скопированный ключ новой строкой. Сохраните и задайте права:
```bash
chmod 600 ~/.ssh/authorized_keys
```

### 3.6. Создание демона автоподключения туннеля
В HA через **File Editor** создайте исполняемый скрипт `/config/tunnel.sh`:
```bash
#!/bin/bash
chmod 600 /config/openvpn/id_dacha
while true; do
    echo "Запуск SSH-туннеля..."
    ssh -i /config/openvpn/id_dacha \
        -o StrictHostKeyChecking=no \
        -o ConnectTimeout=10 \
        -o ServerAliveInterval=15 \
        -o ServerAliveCountMax=3 \
        -N -R 8080:192.168.2.113:80 \
        root@95.100.200.50
    echo "Связь потеряна. Переподключение через 10 секунд..."
    sleep 10
done
```

### 3.7. Настройка автозапуска в Advanced SSH Add-on
Откройте **Настройки** -> **Дополнения** -> **Advanced SSH & Web Terminal** -> **Конфигурация** (переключите в режим YAML через три точки):
```yaml
ssh:
  username: hassio
  password: ВАШ_СЛОЖНЫЙ_ПАРОЛЬ
  authorized_keys: []
  sftp: false
  compatibility_mode: false
  allow_agent_forwarding: true
  allow_remote_port_forwarding: true
  allow_tcp_forwarding: true
zsh: true
share_sessions: false
packages: []
init_commands:
  - chmod +x /config/tunnel.sh
  - nohup /config/tunnel.sh > /dev/null 2>&1 &
```
Нажмите **Сохранить** и **Перезапустить** дополнение.

### 3.8. Конфигурация ядра Home Assistant
Откройте `/config/configuration.yaml` и приведите к чистому виду:
```yaml
default_config:

frontend:
  themes: !include_dir_merge_named themes

automation: !include automations.yaml
script: !include scripts.yaml
scene: !include scenes.yaml

http:
  server_port: 80

homeassistant:
  external_url: https://dacha.djezzy.ru
```
Сохраните файл и выполните полный перезапуск HA (**Настройки** -> **Система** -> кнопка **Перезапуск** в верхнем углу).

---

## ЧАСТЬ 4. Проверка работы

1. Подождите 1 минуту после перезагрузки.
2. Откройте в браузере `https://dacha.djezzy.ru`.
3. Система откроется с валидным SSL-сертификатом от Let's Encrypt.
4. При обрыве связи или отключении питания туннель автоматически переподключится в течение 10–15 секунд после появления сети.

# Миграция настроенного узла HAOS в другую подсеть

Инструкция по быстрому переводу готового сервера Home Assistant из одной локальной сети в другую (например, переезд из городской сети `192.168.0.201` в деревенскую сеть `192.168.2.113`).

---

## 📌 Что меняется

- **Старый локальный IP:** `192.168.0.201` (подсеть `192.168.0.0/24`)
- **Новый локальный IP:** `192.168.2.113` (подсеть `192.168.2.0/24`)
- **VPS и Домен:** Остаются прежними (`45.142.36.51` / `home.djezzy.ru`).

---

## ШАГ 1. Сетевые настройки на новом месте

1. Подключите сервер HA к новому роутеру проводом Ethernet.
2. В веб-интерфейсе нового роутера зафиксируйте статический IP-адрес для HAOS: `192.168.2.113`.
3. Убедитесь, что сервер доступен внутри локальной сети по адресу:
   ```text
   http://192.168.2.113:80
   ```

---

## ШАГ 2. Корректировка OpenVPN конфигурации на HA

Так как подсеть изменилась, нужно обновить правило фильтрации маршрутов, чтобы клиент OpenVPN не изолировал устройство от новой локалки.

1. Откройте **File Editor** в Home Assistant.
2. Перейдите к файлу `/config/openvpn/ha-client.ovpn`.
3. Найдите строку:
   ```ini
   pull-filter ignore "route 192.168.0.0 255.255.255.0"
   ```
4. Замените её на новую подсеть:
   ```ini
   pull-filter ignore "route 192.168.2.0 255.255.255.0"
   ```
5. Сохраните файл.
6. Перейдите в **Настройки** -> **Дополнения** -> **OpenVPN Client** и нажмите **Перезапустить** (Restart).

---

## ШАГ 3. Обновление скрипта SSH-туннеля

Необходимо указать новый целевой IP-адрес для проброса порта.

1. Откройте файл `/config/tunnel.sh` через **File Editor**.
2. Найдите блок вызова `ssh`:
   ```bash
   -N -R 8080:192.168.0.201:80 \
   ```
3. Замените адрес на новый:
   ```bash
   -N -R 8080:192.168.2.113:80 \
   ```
4. Сохраните файл.
5. Перейдите в **Настройки** -> **Дополнения** -> **Advanced SSH & Web Terminal** и нажмите **Перезапустить** (Restart).

---

## ШАГ 4. Сброс зависшего порта на VPS (Один раз)

При смене IP старая сессия на стороне VPS может удерживать порт 8080 в течение пары минут. Чтобы ускорить переподключение:

1. Зайдите на VPS:
   ```bash
   ssh root@45.142.36.51
   ```
2. Принудительно освободите порт:
   ```bash
   fuser -k 8080/tcp
   ```

---

## ШАГ 5. Финальная перезагрузка и проверка

1. В Home Assistant откройте: **Настройки** -> **Система** -> кнопка **Перезапуск** в правом верхнем углу.
2. Подождите 1 минуту.
3. Проверьте статус туннеля на VPS:
   ```bash
   curl -I http://localhost:8080
   ```
   *(Должен вернуться HTTP-ответ `200` или `405`)*.
4. Откройте адрес в браузере:
   ```text
   https://home.djezzy.ru
   ```

---

## ⚡ Экспресс-таблица изменений

| Файл | Параметр | Было | Стало |
| :--- | :--- | :--- | :--- |
| `/config/openvpn/ha-client.ovpn` | Фильтр маршрута | `192.168.0.0` | `192.168.2.0` |
| `/config/tunnel.sh` | Назначение туннеля | `192.168.0.201:80` | `192.168.2.113:80` |
