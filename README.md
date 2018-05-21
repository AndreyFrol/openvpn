1. sudo apt-get update
2. sudo apt-get install openvpn easy-rsa //устанавливаем openvpn, а также easy-rsa, который позволит настроить внутренний центр сертификации для использования с нашей VPN
Далее создаем собственный центр сертификации для выпуска сертификатов (шифрование трафика между сервером и клиентом)
3. make-cadir ~/openvpn-ca // копируем шаблонную easy-rsa в домашнюю директорию
4. cd ~/openvpn-ca
5. vi vars //редактируем переменные центра сертификации
export KEY_COUNTRY="RU"
export KEY_PROVINCE="MO"
export KEY_CITY="Moscow"
export KEY_ORG="DigitalOcean"
export KEY_EMAIL="sofamen0212@gmail.com"
export KEY_OU="Community"

export KEY_NAME="server"
6. cd ~/openvpn-ca
7. source vars
8. ./clean-all //система сделает: rm -rf on /home/USER!!!/openvpn-ca/keys
9. ./build-ca //создаем корневой центр сертификации. Команда запустит процесс создания ключа сертификата корневого центра сертификации. Поскольку мы задали все переменные в файле vars, все необходимые значения будут введены автоматически. Нажимайте ENTER для подтверждения выбора.
10. ./build-key-server server //создания сертификата OpenVPN и ключей для сервера.
Согласитесь со всеми значениями по умолчанию, нажимая ENTER. Не задавайте challenge password. В конце процесса два раза введите y для подписи и подтверждения создания сертификата
11. ./build-dh //генерируем сильные ключи протокола Диффи-Хеллмана, используемые при обмене ключами, командой
12. openvpn --genkey --secret keys/ta.key //генерируем подпись HMAC для усиления способности сервера проверять целостность TSL
____________________________________________________________________________________
Далее генерируем сертификат и пару ключей для клиента
13. cd ~/openvpn-ca
14. source vars
15. ./build-key client1 //создание файлов без пароля для облегчения автоматических соединений
15'. ./build-key-pass client1//Для создания файлов, защищённых паролем
16. cd ~/openvpn-ca/keys
17. sudo cp ca.crt ca.key server.crt server.key ta.key dh2048.pem /etc/openvpn //копируем нужные нам файлы в директорию /etc/openvpn (сертификат, ключ центра сертификации, сертификат и ключ сервера, подпись HMAC и файл Diffie-Hellman)
18. gunzip -c /usr/share/doc/openvpn/examples/sample-config-files/server.conf.gz | sudo tee /etc/openvpn/server.conf // копируем и распаковываем файл-пример конфигурации OpenVPN в конфигурационную директорию, мы будем использовать этот файл в качестве базы для наших настроек
____________________________________________________________________________________
Далее занимаемся настройкой конфигурационного файла сервера
19. sudo nano /etc/openvpn/server.conf
Базовая настройка
Сначала найдём секцию HMAC поиском директивы tls-auth. Удалите ";" для того, чтобы раскомментировать строку с tls-auth. Далее добавьте параметр key-direction и установите его значение в "0".
tls-auth ta.key 0 # This file is secret
key-direction 0
Далее найдём секцию шифрования, нас интересуют закомментированные строки cipher. Шифр AES-128-CBC обеспечивает хороший уровень шифрования и широко поддерживается другими программными продуктами. Удалите ";" для раскомментирования строки AES-128-CBC. Под этой строкой добавьте строку auth и выберите алгоритм HMAC. Хорошим выбором будет SHA256.
cipher AES-128-CBC
auth SHA256
Наконец, найдите настройки user и group и удалите ";" для раскомментирования этих строк:
user nobody
group nogroup

Сделанные нами настройки создают VPN соединение между двумя машинами, но они не заставляют эти машины использовать VPN соединение. Если вы хотите использовать VPN соединение для всего своего трафика, вам необходимо протолкнуть (push) настройки DNS на клиентские машины.

Для этого вам необходимо раскомментировать несколько директив. Найдите секцию redirect-gateway и удалите ";" из начала строки для расскоментирования redirect-gateway:

push "redirect-gateway def1 bypass-dhcp"

Чуть ниже находится секция dhcp-option. Удалите ";" для обеих строк:

push "dhcp-option DNS 208.67.222.222"
push "dhcp-option DNS 208.67.220.220"
Это позволит клиентам сконфигурировать свои настройки DNS для использования VPN соединения в качестве основного.

Настройка перенаправления IP
Сначала разрешим серверу перенаправлять трафик. Это ключевая функциональность нашего VPN сервера
20. vi /etc/sysctl.conf //Найдите строку настройки net.ipv4.ip_forward. Удалите "#" из начала строки, чтобы раскомментировать её
21. sudo sysctl -p //для приминения настроек к текущей сессии

Настройка UFW для сокрытия соединений клиентов
22. ip route | grep default //находим публичный интерфейс
23. vi /etc/ufw/before.rules
#
# rules.before
#
# Rules that should be run before the ufw command line added rules. Custom
# rules should be added to one of these chains:
#   ufw-before-input
#   ufw-before-output
#   ufw-before-forward
#

# START OPENVPN RULES
# NAT table rules
*nat
:POSTROUTING ACCEPT [0:0] 
# Allow traffic from OpenVPN client to eth0
-A POSTROUTING -s 10.8.0.0/8 -o eth0 -j MASQUERADE
COMMIT
# END OPENVPN RULES

# Don't delete these required lines, otherwise there will be errors
*filter

добавляем таблицу nat

24. vi /etc/default/ufw //DEFAULT_FORWARD_POLICY="ACCEPT" сообщаем UFW, что ему по умолчанию необходимо разрешать перенаправленные пакеты
25. sudo ufw allow 1194/udp
26. sudo ufw allow OpenSSH
27. sudo ufw disable
28. sudo ufw enable
29. sudo systemctl start openvpn@server //запускаем сервер OpenVPN
30. sudo systemctl status openvpn@server //проверяем
31. sudo systemctl enable openvpn@server //автоматическое включение при загрузке сервера
32. mkdir -p ~/client-configs/files //В домашней директории создайте структуру директорий для хранения файлов
33. chmod 700 ~/client-configs/files //права доступа для созданных директорий
34. cp /usr/share/doc/openvpn/examples/sample-config-files/client.conf ~/client-configs/base.conf //скопируем конфигурацию-пример в нашу директорию для использования в качестве нашей базовой конфигурации
35. vi ~/client-configs/base.conf
# The hostname/IP and port of the server.
# You can have multiple remote entries
# to load balance between the servers.
remote server_IP_address 1194

proto udp

# Downgrade privileges after initialization (non-Windows only)
user nobody
group nogroup

# SSL/TLS parms.
# See the server config file for more
# description.  It's best to use
# a separate .crt/.key file pair
# for each client.  A single ca
# file can be used for all clients.
#ca ca.crt        //закоментировать
#cert client.crt  //закоментировать
#key client.key   //закоментировать

cipher AES-128-CBC //добавить строку
auth SHA256        //добавить строку

key-direction 1    //добавить строку

# script-security 2                     //добавить эти
# up /etc/openvpn/update-resolv-conf    //строки
# down /etc/openvpn/update-resolv-conf  //для Linux

Теперь создадим простой скрипт для генерации файлов конфигурации с релевантными сертификатами, ключами и файлами шифрования. Он будет помещать сгенерированные файла конфигурации в директорию ~/client-configs/files.
36. vi ~/client-configs/make_config.sh  //создаем файл и вставляем скрипт

#!/bin/bash

# First argument: Client identifier

KEY_DIR=~/openvpn-ca/keys
OUTPUT_DIR=~/client-configs/files
BASE_CONFIG=~/client-configs/base.conf

cat ${BASE_CONFIG} \
    <(echo -e '<ca>') \
    ${KEY_DIR}/ca.crt \
    <(echo -e '</ca>\n<cert>') \
    ${KEY_DIR}/${1}.crt \
    <(echo -e '</cert>\n<key>') \
    ${KEY_DIR}/${1}.key \
    <(echo -e '</key>\n<tls-auth>') \
    ${KEY_DIR}/ta.key \
    <(echo -e '</tls-auth>') \
    > ${OUTPUT_DIR}/${1}.ovpn

37. chmod 700 ~/client-configs/make_config.sh
38. cd ~/client-configs
39. ./make_config.sh client1
40. ls ~/client-configs/files //проверка
41. sftp sammy@openvpn_server_ip:client-configs/files/client1.ovpn ~/ //доставка конфигураций клиентам


описание: https://www.digitalocean.com/community/tutorials/openvpn-ubuntu-16-04-ru#шаг-10-создание-инфраструктуры-настройки-клиентов
