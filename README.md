# Wireguard-server-setup
Описание настройки Wireguard-сервера на Ubuntu
## Настройка доступа к удаленному серверу
Чтобы зайти на удаленный сервер используйте команду:
```bash
ssh root@<ip>
```
После этого выходим с удаленного сервера и копируем на него ssh-ключи:
```bash
ssh-copy-id root@<ip>
```
Заходим на удаленный сервер и в настройках ssh запрещаем вход по паролю, чтобы зайти можно было только при наличии ssh-ключа
```bash
vim /etc/ssh/sshd_config
```
Находим строчку `#PasswordAuthentication yes` и меняем на `#PasswordAuthentication no` и перегружаем ssh:
```bash
service ssh restart
```
Обновляем все что есть на сервере:
```bash
apt update && apt upgrade -y
```
## Установка wireguard
```bash
apt install wireguard -y
```
## Конфигурация wireguard
Заходим в директорию wireguard
```bash
cd /etc/wireguard/
```
Генерим ключи сервера:
```bash
wg genkey | tee /etc/wireguard/privatekey | wg pubkey | tee /etc/wireguard/publickey
```
Проставляем права на приватный ключ:
```bash
chmod 600 /etc/wireguard/privatekey
```
Проверяем название сетевого интерфейса:
```bash
ip a
```
Скорее всего название сетевого интерфейса eth0 - мы будем использовать его при создании файла конфигурации:
```bash
vim /etc/wireguard/wg0.conf
```
Копируем туда эти строчки:
```
[Interface]
PrivateKey = <privatekey>
Address = 10.0.0.1/24
ListenPort = 51830
PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
```
Вместо `<privatekey>` вставляем содержимое ключа `/etc/wireguard/privatekey`
Настраиваем IP форвардинг:
```bash
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p
```
Включаем systemd демон с wireguard:
```bash
systemctl enable wg-quick@wg0.service
systemctl start wg-quick@wg0.service
systemctl status wg-quick@wg0.service
```
## Настраиваем клиента
Создаём ключи клиента:
```bash
wg genkey | tee /etc/wireguard/bahentos_privatekey | wg pubkey | tee /etc/wireguard/bahentos_publickey
```
Добавляем в конфиг сервера клиента
```bash
vim /etc/wireguard/wg0.conf
```
Вставляем туда это:
```
[Peer]
PublicKey = <client-publickey>
AllowedIPs = 10.0.0.2/32
```
Вместо `<client-publickey>` — заменяем на содержимое файла /etc/wireguard/bahentos_publickey

Перезагружаем systemd сервис с wireguard:
```
systemctl restart wg-quick@wg0
systemctl status wg-quick@wg0
```
___
На локальной машине (например, на ноутбуке) создаём текстовый файл с конфигом клиента:
```bash
vim bahentos_wb.conf
```
Содержимое конфига:
```
[Interface]
PrivateKey = <CLIENT-PRIVATE-KEY>
Address = 10.0.0.2/32
DNS = 8.8.8.8

[Peer]
PublicKey = <SERVER-PUBKEY>
Endpoint = <SERVER-IP>:51830
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 20
```
Здесь `<CLIENT-PRIVATE-KEY>` заменяем на приватный ключ клиента, то есть содержимое файла `/etc/wireguard/bahentos_privatekey` на сервере.
  
`<SERVER-PUBKEY>` заменяем на публичный ключ сервера, то есть на содержимое файла `/etc/wireguard/publickey` на сервере. <SERVER-IP> заменяем на IP сервера.

Этот файл открываем в Wireguard клиенте — и жмем в клиенте кнопку подключения.
