# Налаштування власного VPN-сервера за допомогою OpenVPN

## На сервері

### Вхід до сервера

Відкрити термінал `cmd` та за допомогою стандартної утиліти `ssh` підключитись до сервера від імені користувача `root`:

```powershell
ssh root@<ip_сервера>
```

З'явиться запит на підтвердження автентичності хоста — ввести `yes`:

```
The authenticity of host '***.***.***.*** (***.***.***.***)' can't be established.
****** key fingerprint is **************************************************.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
```

Далі з'явиться запит пароля — ввести пароль користувача `root`:

```
root@***.***.***.***'s password:
```

Після успішного підключення відобразиться повідомлення виду:

```
Linux ****************** *.*.*-**-**** #1 SMP PREEMPT_DYNAMIC ...
The programs included with the Debian GNU/Linux system are free software;
...
Last login: **** **** * **:**:** **** from ***.***.***.***
root@dedicate:~#
```

---

### Встановлення компонентів OpenVPN

Оновити пакетний менеджер та встановити необхідні компоненти:

```bash
apt update
apt install openvpn ufw openssl iptables-persistent
```

> `iptables-persistent` знадобиться пізніше для збереження правил firewall між перезавантаженнями.

---

### Генерація сертифікатів та ключів

Створити робочу директорію та перейти до неї:

```bash
mkdir -p /etc/openvpn/server
cd /etc/openvpn/server
```

**Центр сертифікації (CA):**

```bash
openssl genrsa -out ca.key 4096

openssl req -x509 -new -nodes \
  -key ca.key \
  -sha256 \
  -days 3650 \
  -out ca.crt
```

**Ключ та сертифікат сервера:**

```bash
openssl genrsa -out server.key 4096

openssl req -new \
  -key server.key \
  -out server.csr
```

Підписати серверний сертифікат через CA:

```bash
openssl x509 -req \
  -in server.csr \
  -CA ca.crt \
  -CAkey ca.key \
  -CAcreateserial \
  -out server.crt \
  -days 3650 \
  -sha256
```

**Ключ та сертифікат клієнта:**

```bash
openssl genrsa -out client.key 4096

openssl req -new \
  -key client.key \
  -out client.csr
```

Підписати клієнтський сертифікат через CA:

```bash
openssl x509 -req \
  -in client.csr \
  -CA ca.crt \
  -CAkey ca.key \
  -CAcreateserial \
  -out client.crt \
  -days 3650 \
  -sha256
```

**TLS-ключ для додаткового захисту:**

```bash
openvpn --genkey secret ta.key
```

---

### Налаштування конфігурації OpenVPN

Відкрити файл конфігурації сервера для редагування:

```bash
nano /etc/openvpn/server/server.conf
```

Вставити такий вміст, після чого зберегти файл (`Ctrl+X`, потім `Y`, потім `Enter`):

```
port 1194
proto udp
dev tun

ca /etc/openvpn/server/ca.crt
cert /etc/openvpn/server/server.crt
key /etc/openvpn/server/server.key
tls-auth /etc/openvpn/server/ta.key 0

topology subnet
server 10.8.0.0 255.255.255.0

push "redirect-gateway def1 bypass-dhcp"
push "dhcp-option DNS 1.1.1.1"
push "dhcp-option DNS 8.8.8.8"

keepalive 10 60

cipher AES-128-GCM
ncp-ciphers AES-128-GCM:AES-256-GCM
auth SHA256

persist-key
persist-tun

user nobody
group nogroup

verb 3

explicit-exit-notify 1
```

---

### Кінцеве налаштування та запуск сервісу

Увімкнути пересилання пакетів (IP forwarding):

```bash
echo "net.ipv4.ip_forward=1" > /etc/sysctl.d/99-openvpn.conf
sysctl --system
```

Налаштувати NAT для перенаправлення трафіку клієнтів через серверний інтерфейс:

```bash
iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE
netfilter-persistent save
```

> Якщо мережевий інтерфейс сервера називається не `eth0` (наприклад, `ens3` або `enp0s3`), замінити відповідно. Перевірити назву можна командою `ip a`.

Запустити сервіс та додати його до автозапуску:

```bash
systemctl enable openvpn-server@server
systemctl start openvpn-server@server
```

Перевірити статус сервісу:

```bash
systemctl status openvpn-server@server
```

---

## На клієнті

### Перенесення файлів із сервера

На машині клієнта (наприклад, через `scp`) скопіювати необхідні файли із сервера:

```bash
scp root@<ip_сервера>:/etc/openvpn/server/ca.crt ./
scp root@<ip_сервера>:/etc/openvpn/server/client.crt ./
scp root@<ip_сервера>:/etc/openvpn/server/client.key ./
scp root@<ip_сервера>:/etc/openvpn/server/ta.key ./
```

### Створення конфігураційного файлу клієнта

Створити файл `client.ovpn` та вставити до нього наступний вміст, замінивши `<ip_сервера>` на реальну IP-адресу сервера, а вміст тегів — на відповідні дані з раніше скопійованих файлів:

```
client
dev tun
proto udp

remote <ip_сервера> 1194

resolv-retry infinite
nobind

persist-key
persist-tun

remote-cert-tls server

cipher AES-128-GCM
auth SHA256

key-direction 1

verb 3

<ca>
# вміст файлу ca.crt
</ca>

<cert>
# вміст файлу client.crt
</cert>

<key>
# вміст файлу client.key
</key>

<tls-auth>
# вміст файлу ta.key
</tls-auth>
```

### Підключення до VPN

Відкрити клієнт [OpenVPN Connect](https://openvpn.net/client/) або OpenVPN GUI, імпортувати файл `client.ovpn` та натиснути **Connect**.
