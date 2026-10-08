# Умный обход блокировок: OpenVPN Split-Tunneling + Offline DNS

Это полное руководство по развертыванию VPN-схемы, которая изолирует трафик только нужных сайтов и полностью исключает утечки DNS за счет использования локального оффлайн-словаря.

---

## Часть 1. Настройка Сервера

### 1. Подготовка и установка OpenVPN
Мы используем установщик от Nyr, так как он более стабилен на старых версиях OpenVPN.

```bash
# Исправляем PATH (решает проблему "openvpn: command not found" на минимальных ОС)
export PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:$PATH"

# Устанавливаем базовые зависимости
apt-get update && apt-get install -y nginx dnsutils cron curl openvpn

# Скачиваем и запускаем стабильный скрипт установки OpenVPN (от Nyr)
curl -O https://raw.githubusercontent.com/Nyr/openvpn-install/master/openvpn-install.sh
chmod +x openvpn-install.sh
./openvpn-install.sh
# Соглашаемся со всеми настройками (Enter), вводим имя клиента (например, client1).
# Файл client1.ovpn появится в /root/.
```

### 2. Настройка Nginx для раздачи словаря
*Ветка решения:* Nginx падал при попытке привязаться к `10.8.0.1` до запуска OpenVPN. Решение — слушать все интерфейсы, но ограничить доступ через встроенные ACL.

```bash
cat << 'EOF' > /etc/nginx/sites-available/vpn-dict
server {
    listen 80;
    server_name _;
    root /var/www/html;

    # Доступ только для VPN-клиентов и локалхоста
    allow 10.8.0.0/24;
    allow 127.0.0.1;
    deny all;
}
EOF

ln -sf /etc/nginx/sites-available/vpn-dict /etc/nginx/sites-enabled/
rm -f /etc/nginx/sites-enabled/default
systemctl restart nginx
```

### 3. Скрипт генерации словаря
*Ветка решения:* Изначально словарь получался пустым из-за невидимых символов `\r` (переносы строк Windows) в файле `domains.txt`, а утилита `dig` не всегда корректно работала в минимальных образах. Решение: очистка `tr -d '\r'` и использование системного `getent ahosts`.

```bash
mkdir -p /etc/openvpn

# Создаем список доменов (без виндовских переносов строк)
echo -e "twitter.com\napi.twitter.com\ninstagram.com" > /etc/openvpn/domains.txt

# Создаем скрипт-генератор
cat << 'EOF' > /etc/openvpn/build_dict.sh
#!/bin/bash
DICT="/var/www/html/vpn-dictionary.txt"
> $DICT
while read -r domain; do
    domain=$(echo "$domain" | tr -d '\r' | xargs)
    [[ -z "$domain" || "$domain" == \#* ]] && continue
    # Надежный резолв IP-адресов
    ips=$(getent ahosts "$domain" | awk '{print $1}' | grep -E '^[0-9.]+$' | sort -u)
    for ip in $ips; do 
        echo "$ip $domain" >> $DICT
    done
done < /etc/openvpn/domains.txt
EOF

chmod +x /etc/openvpn/build_dict.sh

# Запускаем вручную первый раз
/etc/openvpn/build_dict.sh

# Добавляем автообновление каждый час в cron
systemctl enable cron
systemctl start cron
(crontab -l 2>/dev/null; echo "0 * * * * /etc/openvpn/build_dict.sh") | crontab -
```

---

## Часть 2. Настройка Клиента

### 1. Подготовка и освобождение порта 53
*Ветка решения:* Локальный `dnsmasq` падал при запуске, так как 53 порт был занят встроенным `systemd-resolved`. Решение: отключить системный резолвер.

```bash
# Останавливаем дефолтный резолвер
systemctl stop systemd-resolved
systemctl disable systemd-resolved

# Устанавливаем dnsmasq
apt-get update && apt-get install -y dnsmasq curl

# Замыкаем DNS-запросы системы на саму себя
rm -f /etc/resolv.conf
echo "nameserver 127.0.0.1" > /etc/resolv.conf
```

### 2. Настройка dnsmasq
```bash
touch /etc/vpn-dictionary.txt

cat << 'EOF' > /etc/dnsmasq.conf
# Читаем наш оффлайн-словарь
addn-hosts=/etc/vpn-dictionary.txt

# Игнорируем системный resolv.conf, чтобы не было "петель" (DNS loop)
no-resolv

# Указываем DNS-серверы для обычных сайтов
server=8.8.8.8
server=1.1.1.1
EOF

systemctl restart dnsmasq
```

### 3. Скрипт автомаршрутизации (up.sh)
*Ветка решения:* При быстром старте OpenVPN команда `curl` отваливалась, так как туннель еще не успевал начать пропускать трафик, из-за чего словарь затирался пустотой. Решение: добавлен `sleep 3`.

```bash
mkdir -p /etc/openvpn

cat << 'EOF' > /etc/openvpn/up.sh
#!/bin/bash
# Ждем 3 секунды, чтобы туннель tun0 стабилизировался
sleep 3

# Скачиваем свежий словарь с сервера
curl -s --connect-timeout 10 http://10.8.0.1/vpn-dictionary.txt -o /etc/vpn-dictionary.txt

# Перезапускаем dnsmasq для применения адресов
systemctl restart dnsmasq

# Прописываем маршрут для каждого IP-адреса в туннель
awk '{print $1}' /etc/vpn-dictionary.txt | sort -u | while read ip; do
    ip route add "$ip" dev "$1"
done
EOF

chmod +x /etc/openvpn/up.sh
```

### 4. Подключение (Файл .ovpn)
Откройте файл клиента (например, `client1.ovpn`), перенесенный с сервера, и добавьте в самый конец, вне всех тегов (после `</tls-crypt>`), следующие три строки:

```text
route-nopull
script-security 2
up /etc/openvpn/up.sh
```

**Запуск соединения:**
```bash
# Запуск в фоновом режиме (потребуются права root/sudo)
sudo openvpn --config client1.ovpn --daemon
```

---

## Продвинутая версия: Динамический ipset + Самообучающийся кэш (Smart Learner)

Если заблокированные сайты часто меняют IP-адреса (используют CDN), статический словарь будет за ними не успевать. В этой схеме мы используем динамический словарь (`ipset`) и "умного" демона, который анализирует ответы и сохраняет в постоянный кэш только самые стабильные адреса.

### 1. Переводим Сервер в режим DNS-резолвера
На сервере больше не нужен Nginx. Установим `dnsmasq`, чтобы сервер отвечал на запросы из туннеля:
```bash
apt-get update && apt-get install -y dnsmasq
cat << 'EOF' > /etc/dnsmasq.conf
interface=tun0
bind-interfaces
server=8.8.8.8
server=1.1.1.1
EOF
systemctl restart dnsmasq
```

### 2. Клиент: Настройка dnsmasq и ipset
Устанавливаем `ipset` и обновляем конфигурацию:
```bash
apt-get update && apt-get install -y ipset iptables

cat << 'EOF' > /etc/dnsmasq.conf
no-resolv
server=8.8.8.8 # Обычный интернет

# Запросы для обхода отправляем на VPN-сервер
server=/twitter.com/10.8.0.1
server=/api.twitter.com/10.8.0.1
server=/instagram.com/10.8.0.1

# Записываем полученные IP во временную память (ipset)
ipset=/twitter.com/api.twitter.com/instagram.com/vpn_ips

# Настройки для самообучения
log-queries
log-facility=/var/log/dnsmasq.log
addn-hosts=/etc/vpn-static-dictionary.txt
EOF

touch /etc/vpn-static-dictionary.txt
touch /var/log/dnsmasq.log
systemctl restart dnsmasq
```

### 3. Клиент: Демон Самообучения (Smart Learner)
Скрипт анализирует логи. Если IP-адрес выдается 3 раза, причем между каждым разом проходит минимум 20 минут, адрес признается "стабильным" и сохраняется на диск.

Создайте файл `/usr/local/bin/vpn-learner.sh`:
```bash
cat << 'EOF' > /usr/local/bin/vpn-learner.sh
#!/bin/bash
LOG="/var/log/dnsmasq.log"
DICT="/etc/vpn-static-dictionary.txt"
IPSET_SAVE="/etc/vpn_ips.save"

tail -F $LOG | awk -v dict="$DICT" -v save="$IPSET_SAVE" '
BEGIN {
    while ((getline line < dict) > 0) {
        split(line, a, " ");
        if (a[1] != "") known[a[2] "-" a[1]] = 1;
    }
    close(dict)
}
/reply/ && /is/ {
    domain = ""
    ip = ""
    for (i=1; i<=NF; i++) {
        if ($i == "reply" && $(i+2) == "is") {
            domain = $(i+1)
            ip = $(i+3)
            break
        }
    }
    if (domain != "" && ip ~ /^[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+$/) {
        key = domain "-" ip
        if (!known[key]) {
            now = systime()
            if (count[key] == 0) {
                count[key] = 1
                last_seen[key] = now
            } else if (now - last_seen[key] >= 1200) {
                count[key]++
                last_seen[key] = now
                if (count[key] == 3) {
                    known[key] = 1
                    print ip " " domain >> dict
                    system("ipset save vpn_ips > " save)
                    system("systemctl reload dnsmasq")
                }
            }
        }
    }
}'
EOF
chmod +x /usr/local/bin/vpn-learner.sh
```

Создайте службу `/etc/systemd/system/vpn-learner.service`:
```bash
cat << 'EOF' > /etc/systemd/system/vpn-learner.service
[Unit]
Description=Smart VPN DNS Learner
After=dnsmasq.service

[Service]
ExecStart=/usr/local/bin/vpn-learner.sh
Restart=always

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload && systemctl enable --now vpn-learner
```

### 4. Клиент: Обновление up.sh
Обновите `/etc/openvpn/up.sh` для восстановления сохраненных маршрутов:
```bash
cat << 'EOF' > /etc/openvpn/up.sh
#!/bin/bash
if [ -f /etc/vpn_ips.save ]; then
    ipset restore < /etc/vpn_ips.save
else
    ipset create vpn_ips hash:ip 2>/dev/null
fi

iptables -t mangle -C OUTPUT -m set --match-set vpn_ips dst -j MARK --set-mark 1 2>/dev/null || \
iptables -t mangle -A OUTPUT -m set --match-set vpn_ips dst -j MARK --set-mark 1

ip rule del fwmark 1 table 100 2>/dev/null
ip rule add fwmark 1 table 100
ip route replace default dev $1 table 100

systemctl restart dnsmasq
EOF
```
