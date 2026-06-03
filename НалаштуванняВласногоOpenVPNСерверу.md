# Налаштування власного VPN серверу за домогою клієнта OpenVPN
## На сервері
### Вхід до серверу
Відкрити термінал `cmd` та, за допогомою стандартної утиліти `ssh`, підключитись до серверу від імені користувача `root`:
```powershell
ssh root@<ip_серверу>
```
З'явиться текст із підтвердженням, де необхідно вказати `yes`:
```
The authenticity of host '***.***.***.*** (***.***.***.***)' can't be established.
****** key fingerprint is **************************************************.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
```
І далі буде пропозиція введення паролю, вказати відповідний пароль за користувачем `root`:
```
root@***.***.***.***s password:
```
Після успішного підключення повинно з'явитись подібне повідомлення:
```
Linux ****************** *.*.*-**-**** #1 SMP PREEMPT_DYNAMIC ****** *.*.***-* (****-**-**) x**_**
The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.
Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: **** ****  * **:**:** **** from ***.***.***.***
root@dedicate:~#
```
### Налаштування системи доступу OpenVPN
Оновити пакетний менеджер та через нього встановити відповідні компоненти на сервері:
```bash
apt update
apt install openvpn ufw openssl
```
Створення необхідних директорій та перехід туди:
```
mkdir -p /etc/openvpn/server
cd /etc/openvpn/server
```
Виконати генерацію сертифікатів:
```bash
openssl genrsa -out ca.key 4096

openssl req -x509 -new -nodes \
  -key ca.key \
  -sha256 \
  -days 3650 \
  -out ca.crt
```
Виконати генерацію ключа для серверу:
```bash
openssl genrsa -out server.key 4096

oepnssl req -new \
  -key server.key \
  -out server.csr
```
Підписати серверний сертифікат:
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
Зробити генерацію ключа та сертифікату клієнта:
```bash
openssl genrsa -out client.key 4096

openssal req -new \
  -key client.key \
  -out client.csr
```
Підписати все нові файли:
```bash
openssal x509 -req \
  -in client.csr \
  -CA ca.crt \
  -CAkey ca.key \
  -CAcreateserial \
  -out client.crt \
  -days 3650 \
  -sha256
```
Сформувати TSL ключ:
```bash
openvpn --genkey secret ta.key
```
### Налаштування OpenVPN конфігу
Перейти до редагування серверного конфігу:
```bash
nano /etc/openvpn/server/server.conf
```
В цьому файлі потрібно вставити такі налаштування, після чого `Ctrl + X` і клавіша `Y`:
```bash
port 1194
proto udp
dev tun

ca ca.crt
cert server.crt
key server.key
tls-auth ta.key 0

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
### Кінцеве налаштування та запуск сервісу
Налаштувати NAT підключення з боку серверу:
```bash
echo "net.ipv4.ip_forward=1" > /etc/sysctl.d/99-openvpn.conf
sysctl --system
```
```bash
iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE
netfilter-persistent save
```
Запуск сервісу:
```bash
systemctl enable openvpn-server@server
systemctl start openvpn-server@server
```
Перевірка статусу сервісу:
```bash
systemctl status openvpn-server@server
```
## На клієнті
### Налаштування конфігу сервісу
Перенесення потрібних даних на свій комп'ютер:
```
```
```bash
client
dev tun
proto udp

remote YOUR_SERVER_IP 1194

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
</ca>

<cert>
</cert>

<key>
</key>

<tls-auth>
</tls-auth>
```
