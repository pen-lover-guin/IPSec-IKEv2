# IPSec-IKEv2
How to setup IPSec IKEv2

Server - Ubuntu 18.04

Client - Ubuntu 18.04

[Шаг 1. Установка StrongSwan](#шаг-1-установка-strongswan)

[Шаг 2. Создание центра сертификации](#шаг-2-создание-центра-сертификации)

[Шаг 3. Генерирование сертификата для VPN сервера](#шаг-3-генерирование-сертификата-для-vpn-сервера)

[Шаг 4. Настройка StrongSwan](#шаг-4-настройка-strongswan)

[Шаг 5. Настройка VPN аутентификации](#шаг-5-настройка-vpn-аутентификации)

[Шаг 6. Настойка брандмауэра и IP forwarding](#шаг-6-настойка-брандмауэра-и-ip-forwarding)

[Шаг 7. Настройка клиентов](#шаг-7-настройка-клиентов)

## Шаг 1. Установка StrongSwan

Во-первых, мы установим StrongSwan, демон IPSec с открытым исходным кодом, который мы настроим как наш VPN-сервер. Мы также установим компонент инфраструктуры открытого ключа, чтобы мы могли создать центр сертификации.

Обновим пакеты и установим нужные компоненты:

    sudo apt update
    sudo apt install strongswan strongswan-pki
    
## Шаг 2. Создание центра сертификации

Для идентификации клиентов серверу IKEv2 нужны сертификаты. Поэтому strongswan-pki поставляется с утилитой для генерирования центра сертификации и сертификатов сервера. Для начала создадим каталог для хранения всех этих компонентов. Заблокируем доступ к этому каталогу.

    mkdir -p ~/pki/{cacerts,certs,private}
    chmod 700 ~/pki
    
Теперь у нас есть отдельный каталог для сертификатов. Сгенерируем root ключ – 4096-битный RSA-ключ, с помощью которого вы сможете подписать root-сертификат.

    ipsec pki --gen --type rsa --size 4096 --outform pem > ~/pki/private/ca-key.pem
    
Теперь можно создать ЦС и использовать ключ для подписи сертификата root:

    ipsec pki --self --ca --lifetime 3650 --in ~/pki/private/ca-key.pem \
    --type rsa --dn "CN=VPN root CA" --outform pem > ~/pki/cacerts/ca-cert.pem

Мы можем изменить параметры флага —dn (distinguished name) и указать свою страну, организацию и имя.

## Шаг 3. Генерирование сертификата для VPN сервера

Создадим сертификат и ключ для сервера VPN, с помощью которого клиенты смогут проверить его подлинность.

Для начала создадим закрытый ключ для VPN сервера:

    ipsec pki --gen --type rsa --size 4096 --outform pem > ~/pki/private/server-key.pem
    
Затем создадим сертификат для VPN-сервера и подпишем его с помощью ЦС, который мы получили в предыдущем разделе. Запустим следующий набор команд, предварительно указав в параметрах CN (Common Name) и SAN (Subject Alternate Name) доменное имя или IP-адрес сервера.

    ipsec pki --pub --in ~/pki/private/server-key.pem --type rsa \
        | ipsec pki --issue --lifetime 1825 \
            --cacert ~/pki/cacerts/ca-cert.pem \
            --cakey ~/pki/private/ca-key.pem \
            --dn "CN=server_domain_or_IP" --san "server_domain_or_IP" \
            --flag serverAuth --flag ikeIntermediate --outform pem \
        >  ~/pki/certs/server-cert.pem

Теперь, когда мы сгенерировали все файлы TLS / SSL, которые нужны StrongSwan, мы можем переместить их нв /etc/ipsec.d:

    sudo cp -r ~/pki/* /etc/ipsec.d/

Теперь у нас есть сертификаты, которые защитят взаимодействие клиента и сервера. Кроме того, с их помощью клиенты смогут подтвердить подлинность VPN-сервера.

## Шаг 4. Настройка StrongSwan

У StrongSwan есть стандартный конфигурационный файл. Прежде чем внести какие-либо изменения, создадим его резервную копию, чтобы иметь доступ к параметрам по умолчанию, если что-то пойдет не так:

    sudo mv /etc/ipsec.conf{,.original}
    
Создадим новый файл:

    sudo nano /etc/ipsec.conf
    
Настроим регистрацию состояний демона (это пригодится при отладке и устранении неполадок) и разрешим резервные соединения:

    config setup
        charondebug="ike 1, knl 1, cfg 0"
        uniqueids=no
        
Затем мы создадим раздел конфигурации для нашего VPN. Мы также сообщим StrongSwan создать VPN-туннели IKEv2 и автоматически загружать этот раздел конфигурации при запуске. Добавьте следующие строки в файл:

    . . .
    conn ikev2-vpn
        auto=add
        compress=no
        type=tunnel
        keyexchange=ikev2
        fragmentation=yes
        forceencaps=yes
        
Мы также настроим обнаружение мертвых узлов, чтобы очистить любые «висячие» подключения в случае неожиданного отключения клиента. Добавьте эти строки:

    . . .
    conn ikev2-vpn
        . . .
        dpdaction=clear
        dpddelay=300s
        rekey=no
        
Затем мы настроим параметры IPSec сервера:

    . . .
    conn ikev2-vpn
        . . .
        left=%any
        leftid=@server_domain_or_IP
        leftcert=server-cert.pem
        leftsendcert=always
        leftsubnet=0.0.0.0/0
        
>Примечание. При настройке идентификатора сервера (leftid) используйте символ @, если ваш VPN-сервер будет идентифицирован по имени домена:

>       leftid=@vpn.example.com

>Если сервер будет идентифицирован по его IP-адресу, просто введите IP-адрес:

>       leftid=203.0.113.7

Затем мы можем настроить параметры IPSec на стороне клиента, такие как диапазоны частных IP-адресов и DNS-сервера:
    
    . . .
    conn ikev2-vpn
        . . .
        right=%any
        rightid=%any
        rightauth=eap-mschapv2
        rightsourceip=10.101.10.0/24
        rightdns=8.8.8.8,8.8.4.4
        rightsendcert=never
        
Наконец, мы сообщим StrongSwan запросить у клиента учетные данные пользователя при подключении:

    . . .
    conn ikev2-vpn
        . . .
        eap_identity=%identity

Файл конфигурации должен выглядеть так:

    config setup
        charondebug="ike 1, knl 1, cfg 0"
        uniqueids=no
    
    conn ikev2-vpn
        auto=add
        compress=no
        type=tunnel
        keyexchange=ikev2
        fragmentation=yes
        forceencaps=yes
        dpdaction=clear
        dpddelay=300s
        rekey=no
        left=%any
        leftid=@server_domain_or_IP
        leftcert=server-cert.pem
        leftsendcert=always
        leftsubnet=0.0.0.0/0
        right=%any
        rightid=%any
        rightauth=eap-mschapv2
        rightsourceip=10.101.10.0/24
        rightdns=8.8.8.8,8.8.4.4
        rightsendcert=never
        eap_identity=%identity
        
>Не забудьте заменить @server_domain_or_IP на IP адрес сервера.

Сохраните и закройте файл, как только вы убедитесь, что настроили все как показано.

## Шаг 5. Настройка VPN аутентификации

Наш VPN-сервер теперь настроен на прием клиентских подключений, но у нас еще не настроены учетные данные. Нам нужно сконфигурировать файл ipsec.secrets:

* Нам нужно сообщить StrongSwan, где найти закрытый ключ для нашего сертификата сервера, чтобы сервер мог проходить проверку подлинности для клиентов.

* Нам также необходимо настроить список пользователей, которым будет разрешено подключаться к VPN.

Для начала отредактируем файл:

    sudo nano /etc/ipsec.secrets
    
Сначала мы сообщим StrongSwan, где найти наш закрытый ключ:

    : RSA "server-key.pem"
    
Затем мы определим учетные данные пользователя:

    your_username : EAP "your_password"
    
Теперь, когда мы закончили работу с параметрами VPN, мы перезапустим службу VPN, чтобы применить нашу конфигурацию:

    sudo systemctl restart strongswan
    
Теперь, когда VPN-сервер полностью настроен пришло время перейти к настройке наиболее важной части: настройке firewall.

## Шаг 6. Настойка брандмауэра и IP forwarding

Если вы еще не настроили UFW, вы можете создать базовую конфигурацию и включить ее:

    sudo ufw allow OpenSSH
    sudo ufw enable
    
Теперь добавьте правило, разрешающее UDP-трафик на стандартные порты IPSec, 500 и 4500:

    sudo ufw allow 500,4500/udp
    
Далее мы откроем один из файлов конфигурации UFW, чтобы добавить несколько низкоуровневых политик для маршрутизации и пересылки пакетов IPSec.Перед тем, как изменить этот файл, мы должны найти публичный интерфейс сети (public network interface). Для этого наберите команду:

    ip route | grep default
    
Публичный интерфейс должен следовать за словом “dev”. Например, в нашем случае этот интерфейс называется eth0.

Когда вы нашли публичный интерфейс, откройте файл ```/etc/ufw/before.rules``` в текстовом редакторе:

    sudo nano /etc/ufw/before.rules
    
В верхней части файла (перед строкой ```*filter```) добавьте следующий блок конфигурации:

    *nat
    -A POSTROUTING -s 10.101.10.0/24 -o eth0 -m policy --pol ipsec --dir out -j ACCEPT
    -A POSTROUTING -s 10.101.10.0/24 -o eth0 -j MASQUERADE
    COMMIT
    
    *mangle
    -A FORWARD --match policy --pol ipsec --dir in -s 10.101.10.0/24 -o eth0 -p tcp -m tcp --tcp-flags SYN,RST SYN -m tcpmss --mss 1361:1536 -j TCPMSS --set-mss 1360
    COMMIT
    
    *filter
    :ufw-before-input - [0:0]
    :ufw-before-output - [0:0]
    :ufw-before-forward - [0:0]
    :ufw-not-local - [0:0]
    . . .

Измените eth0 в приведенной выше конфигурации на тот, который вы нашли с помощью ```ip route```.

Далее, после блока ```*filter``` добавьте еще один блок конфигурации:

    . . .
    *filter
    :ufw-before-input - [0:0]
    :ufw-before-output - [0:0]
    :ufw-before-forward - [0:0]
    :ufw-not-local - [0:0]
    
    -A ufw-before-forward --match policy --pol ipsec --dir in --proto esp -s 10.101.10.0/24 -j ACCEPT
    -A ufw-before-forward --match policy --pol ipsec --dir out --proto esp -d 10.101.10.0/24 -j ACCEPT

Прежде чем перезапустить брандмауэр, мы изменим некоторые параметры сетевого ядра, чтобы разрешить маршрутизацию от одного интерфейса к другому. Откроем файл конфигурации параметров ядра UFW:

    sudo nano /etc/ufw/sysctl.conf
    
Нам нужно настроить несколько вещей:

* Во-первых, мы включим пересылку пакетов IPv4.
* Мы отключим обнаружение Path MTU для предотвращения проблем фрагментации пакетов.
* Мы также не принимаем перенаправления ICMP и не отправляем перенаправления ICMP для предотвращения атак man-in-the-middle.

```
. . .

# Включить пересылку пакетов
# Раскоментировать следующую строчку
net/ipv4/ip_forward=1

. . .

# Не принимать ICMP перенаправления
# Убедитесь, что установлено следующее значение
net/ipv4/conf/all/accept_redirects=0

# Не отправлять перенаправления ICMP (мы не маршрутизатор) 
# Добавьте следующие строки
net/ipv4/conf/all/send_redirects=0
net/ipv4/ip_no_pmtu_disc=1
```

Для применения настроек наберите команду:

    sudo sysctl -p

Теперь мы можем применить все наши изменения, отключив и повторно включив брандмауэр:

    sudo ufw disable
    sudo ufw enable

## Шаг 7. Настройка клиентов

Первым делом мы должны скопировать сертификат сервера на клиентское устройство. 

Для передачи файла воспользуемся командой SCP.

    scp user@<ip сервера>:/etc/ipsec.d/cacerts/ca-cert.pem ~/

Далее установим необходимые пакеты:
    
    sudo apt update
    sudo apt install charon-cmd libcharon-extra-plugins

Перейдем в папку, где содержится скаченный ```ca-cert.pem```.

Подключитесь к VPN-серверу с помощью charon-cmd, используя сертификат CA сервера, IP-адрес VPN-сервера и настроенное вами имя пользователя:

    sudo charon-cmd --cert ca-cert.pem --host vpn_domain_or_IP --identity your_username
    
При подключении надо указать пароль пользователя VPN.

Теперь мы должны быть подключены к VPN. Для отключения надо нажать CTRL + C и дождаться закрытия соединения.
