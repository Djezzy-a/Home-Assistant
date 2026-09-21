# Home-Assistant
Гайд по переносу с одной подсети в другую. И шпора для развертывания с нуля
 Полная настройка нового сервера Home Assistant с VPS

Пошаговая инструкция для развёртывания Home Assistant с доступом через HTTPS (домен) на новом VPS и новом сервере.
Исходные данные (пример)

    VPS IP: 95.100.200.50
    Домен: dacha.djezzy.ru
    IP Home Assistant в локальной сети: 192.168.2.113
    Порт HA: 80

ЧАСТЬ 1. Настройка VPS (новый сервер)
1. Подключись к VPS по SSH

Bash

ssh root@95.100.200.50

2. Обнови систему

Bash

apt update && apt upgrade -y

3. Установи OpenVPN сервер

Bash

wget https://git.io/vpn -O openvpn-install.sh
chmod +x openvpn-install.sh
./openvpn-install.sh

Ответь на вопросы:
Вопрос	Ответ
IP address	Enter (автоматически)
Public IPv4 or hostname	Enter
Protocol	1 (UDP)
Port	1194 (Enter)
DNS	1 или 3
Client name	dacha-client
Passwordless client	1 (да)

Скрипт создаст /root/dacha-client.ovpn.
4. Включи IP forwarding

Bash

echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p

5. Установи Caddy

Bash

apt install -y debian-keyring debian-archive-keyring apt-transport-https curl
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | tee /etc/apt/sources.list.d/caddy-stable.list
apt update
apt install -y caddy
systemctl enable caddy

6. Настрой Caddy

Bash

nano /etc/caddy/Caddyfile

Вставь:

caddy

dacha.djezzy.ru {
    reverse_proxy localhost:8080 {
        header_up -X-Forwarded-For
        header_up -X-Forwarded-Proto
        header_up -X-Forwarded-Host
        header_up -X-Real-IP
    }
}

Сохрани (Ctrl+O, Enter, Ctrl+X).

Bash

systemctl restart caddy

7. Настрой SSH-сервер

Bash

nano /etc/ssh/sshd_config

Добавь в конец:

ini

ClientAliveInterval 10
ClientAliveCountMax 3

Сохрани и перезапусти:

Bash

systemctl restart ssh

8. Настрой файрвол (если используется UFW)

Bash

ufw allow 1194/udp
ufw allow 80/tcp
ufw allow 443/tcp
ufw allow 22/tcp
ufw reload

9. Скачай конфиг OpenVPN на локальный компьютер

Bash

scp root@95.100.200.50:/root/dacha-client.ovpn .

ЧАСТЬ 2. Настройка DNS

Зайди в панель управления доменом и добавь A-запись:

text

dacha.djezzy.ru → 95.100.200.50

Подожди 1-5 минут. Проверь:

Bash

ping dacha.djezzy.ru

ЧАСТЬ 3. Настройка Home Assistant
1. Установи OpenVPN Client addon

    Настройки → Дополнения → Магазин дополнений
    Три точки (⋮) → Репозитории
    Добавь:

    text

    https://github.com/rofra/homeassistant-openvpn-client-addon

    Установи OpenVPN Client
    Включи:
        ✅ Запуск при загрузке
        ✅ Ватчдог

2. Установи File Editor addon

Настройки → Дополнения → Магазин дополнений → найди File Editor, установи.
3. Загрузи конфиг OpenVPN

Через File Editor:

    Создай папку /config/openvpn/
    Загрузи файл dacha-client.ovpn
    Открой его и добавь в конец:

ini

pull-filter ignore "redirect-gateway"
pull-filter ignore "route 192.168.2.0 255.255.255.0"

Сохрани.
4. Настрой OpenVPN Client addon

OpenVPN Client → Конфигурация:
Поле	Значение
Path to OpenVPN config	/config/openvpn/dacha-client.ovpn
Password (certificate)	(пусто)
Username	(пусто)
Password	(пусто)
Custom arguments	(пусто)

Сохрани → Запусти.

Проверь Логи — должно быть:

text

Initialization Sequence Completed

5. Установи Advanced SSH & Web Terminal

Настройки → Дополнения → Магазин дополнений → Advanced SSH & Web Terminal.

Включи:

    ✅ Запуск при загрузке
    ✅ Ватчдог

Запусти addon.
6. Создай SSH-ключ

В терминале Advanced SSH:

Bash

ssh-keygen -t ed25519 -f /config/openvpn/id_dacha -N ""
cat /config/openvpn/id_dacha.pub

Скопируй вывод (строка начинается с ssh-ed25519).
7. Добавь ключ на VPS

На VPS:

Bash

nano ~/.ssh/authorized_keys

Вставь ключ новой строкой. Сохрани.

Bash

chmod 600 ~/.ssh/authorized_keys

8. Создай скрипт туннеля

Через File Editor создай /config/tunnel.sh:

Bash

#!/bin/bash
chmod 600 /config/openvpn/id_dacha
while true; do
    echo "Запуск туннеля..."
    ssh -i /config/openvpn/id_dacha \
        -o StrictHostKeyChecking=no \
        -o ConnectTimeout=10 \
        -o ServerAliveInterval=15 \
        -o ServerAliveCountMax=3 \
        -N -R 8080:192.168.2.113:80 \
        root@95.100.200.50
    echo "Туннель упал. Переподключение через 10 секунд..."
    sleep 10
done

Сохрани.
9. Настрой автозапуск туннеля

Advanced SSH & Web Terminal → Конфигурация → Изменить в YAML:

YAML

ssh:
  username: hassio
  password: ВАШ_ПАРОЛЬ
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

Сохрани → Перезагрузи addon.
10. Настрой configuration.yaml

Открой /config/configuration.yaml, замени на:

YAML

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

Сохрани.
11. Перезапусти Home Assistant

Настройки → Система → Перезапуск.
ЧАСТЬ 4. Проверка

Подожди 1 минуту, затем открой:

text

https://dacha.djezzy.ru

Всё должно работать.
