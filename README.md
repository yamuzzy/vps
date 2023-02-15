## Что это?
Это инструкция, как завести свой VPN/VPS. Чтобы не забыть и не потерять.
Все шаги вручную, в результате - свой персональный VPN + графическая "консоль" для добавления новых клиентов.
Disclaimer - сам первый раз пробую, если что не так, дайте знать.

## шаг-1 - заводим себе доменное имя (регистрируем домен)
Этот шаг с duckdns не очень нужен, т.к. wireguard везде хранит ip адреса, а не имена. Получается, что имя нужно только для удобства, чтобы ip-шники не учить.

duckdns.org - бесплатно. Я зарегался от гугла.

надо придумать имя под-домена. Ну не по ip-шнику же заходить!

![ducksns 1st pic](/images/photo_2022-04-09_22-00-32.jpg)

Внимание! duckdns прислал токен в письме. Токен будет нужен потом в скрипте для обновления.

## шаг-2 - завести VPS. 
Я брал https://vdsina.ru. Нужна регистрация, надо закинуть туда чуть денег. Я положил 350 рублей, комиссия была 10 с копейками. Сервер создаётся быстро, а вот письма в гугл (e-mail я указал оттуда) идут медленнее.

![vps 1st pic](/images/photo_2022-04-09_21-44-29.jpg)

- тут просто выбирается самый дешевый. Этого достаточно для VPN. Потом можно увеличивать, если надо.


![vps 2st pic](/images/photo_2022-04-09_21-45-20.jpg)

Шаги по регистрации не привожу, не актуально. Ну или потом.
vsdina даёт один фиксированный ipv4 адрес, так что по сути dyn-dns нам не надо, но всё равно сделаем скрипт, чтобы адрес "обновлялся" и duckdns не протухал.

Когда сервер будет создан (несколько минут), придёт письмо с IP адресом и паролем рута. Я пароль сразу сменил, но, возможно, это и не актуально, т.к. из web-консоли vdsina это тоже можно сделать, если забыли.

Будем заходить от него и ставить-конфигурировать всё. 

Не бойтесь ошибок - хостинг позволяет 

а) ресетнуть сервер, 

б) пересоздать его. При чём, я так понял, что пересоздание или бесплатно, или недорого. Так что, если что-то пойдёт не так, то можно будет начать сначала. У меня получилось с первого раза.

в) с vdsina можно сбросить пароль рута и зайти в vnc. Пока не было нужно, но пробовал, работает.
СРАЗУ! Никаких слабых паролей! Технический долг - было бы здорово прописать, как сделать вход по сертификату... Но я этого не делал. Каждый раз набирать свой пароль уже надоедает... Однако, один раз настроил и больше туда не хожу.

## шаг 3 - начинаем конфигурировать
1. Добавляем своего пользователя и меняем пароль рута
```
adduser  username
```
  Делаем его sudo
```
usermod -aG sudo username
```
Логинимся как username, проверяем, что он работает и у него есть sudo. Обязательно перед тем, как отключаться от рута проверьте в другом окне, что новый account работает нормально!
  
Меняем дефолтный пароль у рута и отлогиниваемся от него.
```
passwd
```
  
## шаг 4 - Ставим собственно wireguard 
Taken mostly from here: https://www.wireguard.com/install/
```
sudo apt install wireguard
```
## шаг 5 - Настройка системы - роутинг должен быть включен в ядре
```
sudo vi /etc/sysctl.conf
```
(кому нравится - используйте nano, конечно!)

В этом файле должна быть добавлена/раскомментирована как минимум одна строка:
```
net.ipv4.ip_forward = 1
```
- Я сделал только одну строку. Остальные не стал (пока). Вот остальные
```
net.ipv6.conf.default.forwarding = 1
net.ipv6.conf.all.forwarding = 1
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.proxy_arp = 0
net.ipv4.conf.default.send_redirects = 1
net.ipv4.conf.all.send_redirects = 0
```  
Then apply the new option with the command below.
```
sudo sysctl -p
```
при этом на экране будет показано, что изменилось.

net.ipv4.ip_forward=1
  
## шаг 6 - Делаем скрипт обновления на duckdns
```
sudo su
mkdir /root/duckdns
vi /root/duckdns/duck.sh
```
Add a line
```
echo url="https://www.duckdns.org/update?domains=ВАШПОДДОМЕН.duckdns.org&token=ТОКЕНОТDUCKDNS" | curl -k -o ~/duckdns/duck.log -K -
```
save the file
```
chmod +x /root/duckdns/duck.sh          
```
и запускаем его
```
.\duck.sh
```
Внезапно обнаружилось, что на vps нету curl, добавляем
```
sudo apt update
apt install curl
```
После этого скрипт работает, надо его зашедулить
```
crontab -e
```
Add a line (обновление dyndns адреса раз в 5 минут)
```
*/5 * * * * ~/duckdns/duck.sh >/dev/null 2>&1
```
(Я потом поменял на раз в 30 минут, тут ничего интересного). Если теперь зайти на duckdns в свой акк, то там будет адрес вашего vps. Ну и на vps теперь можно входить по имени, а не по ip.

## шаг 7 - Защита firewall-ом, если нету. ufw - надстройка над iptables
стандартный файрвол ubuntu - iptables уже стоит. Но им управлять сложно. Ставим надстройку
```
sudo apt install ufw
```  
и конфигурим её
```
sudo ufw allow ssh
sudo ufw allow 8189/udp
sudo ufw allow from 192.168.2.0/24
sudo ufw allow from wg0
sudo ufw default deny incoming
sudo ufw default allow outgoing
ufw default allow routed
```
включаем файрвол
```
sudo ufw enable
sudo ufw status
```
Logging

ufw и iptables очень болтливы. но посмотреть логи бывает очень интересно. Используйте команду tail -f /var/log/имяЛога, чтобы увидеть, что там было в самом конце, получая обновления прямо по ходу дела.
  
To enable logging use:
```
sudo ufw logging on
```
To disable logging use (иначе каждый заблокированный пакет пишется в /var/log/syslog)
```  
sudo ufw logging off
```
Мы разрешили вход только чрезе 22й порт. 

По-идее можно ещё разрешить 5000 для своего IP-шника, хотя бы временно, чтобы зайти на WG-GUI. 

Я по-незнанию всё делал вручную, а потом уже зашел через гуй и страдал, что так оказывается было можно.А потом уже ходил только через клиента.

Как бы я сделал сейчас? Вообще, думаю, порт 22 стоило бы запретить и на него ходить только через web ui vdsina.

## шаг 8 - Install ui for wg remote management 
(if firewall set as said above, it will be available only from VPN.)

Эта удобная штука - мы не будем конфигурироваь WG клиентов руками. Да и сам сервер тоже. А то я вначале задолбался ключи туда-сюда вставлять. гуй доступен по адресу АдресСервера:5000, логин admin/admin

Гуй найден здесь:
  https://github.com/ngoduykhanh/wireguard-ui
  https://github.com/ngoduykhanh/wireguard-ui/releases

```
sudo su
cd /etc/wireguard
wget https://github.com/ngoduykhanh/wireguard-ui/releases/download/v0.3.6/wireguard-ui-v0.3.6-linux-386.tar.gz
  
tar -xzf wireguard…
```
Распаковали

можно скачанный архив удалить. Уже можно запустить гуй через ./scriptname и поиграть, мы настроим, чтобы он сам запускался.


## шаг 9 - конфигурируем wg web ui for remote mgmt
### Create monitoring part for restarting wg service on change
нужен сервис, который будет следить за изменением wg конфигурационного файла и рестартовать wg
```
vi /etc/systemd/system/wgui.service
```
Вставляем
```
[Unit]
Description=Restart WireGuard
After=network.target
[Service]
Type=oneshot
ExecStart=/usr/bin/systemctl restart wg-quick@wg0.service
[Install]
RequiredBy=wgui.path
```
ещё один файл
```
vi /etc/systemd/system/wgui.path
```
вставляем
```
[Unit]
Description=Watch /etc/wireguard/wg0.conf for changes
[Path]
PathModified=/etc/wireguard/wg0.conf
[Install]
WantedBy=multi-user.target
```  
Сервис не стартанёт, если не создать хотя бы пустой cfg file (sudo su, конечно же)
```
touch /etc/wireguard/wg0.conf
```
Apply it
```
systemctl enable wgui.{path,service}
systemctl start wgui.{path,service}
```  
  
### Run wireguard-ui as a service. 
Внимание, после этого сайт wg ui будет виден только изнутри firewall.
```
vi /etc/systemd/system/wireguard-ui.service
```
вставляем
```
[Unit]
Description=Wireguard-UI web site
After=network.target
  
[Service]
Type=simple
WorkingDirectory=/etc/wireguard
ExecStart=/etc/wireguard/wireguard-ui
Restart=always
RestartSec=10
  
[Install]
WantedBy=multi-user.target
```  
Apply it
```
systemctl enable wireguard-ui
systemctl start wireguard-ui
```

## Шаг 10 - Конфигурация автостарта wg на сервере
```
sudo systemctl start wg-quick@wg0
sudo systemctl enable wg-quick@wg0
```
## Шаг 11 - защита порта 22 от взлома
```
sudo apt-get install fail2ban
```  
To configure fail2ban, make a 'local' copy the jail.conf file in /etc/fail2ban 
```  
cd /etc/fail2ban
sudo cp jail.conf jail.local 
```  
Now edit the file: 
```  
sudo vi jail.local 
```
надо игонорировать свои адреса.

1) локалхост ipv4 и ipv6 (второе - по инерции)

2) внутренний адрес VPN. Внимане, это НЕ адрес домашней сети, не адрес рабочей, это своя собственная сеть. Дайте ей такое адресное пространство, чтобы она ни с чем не пересекалась.

3) я добавил те адреса, с которых я буду заходить сам

добавляем
```     
ignoreip = 127.0.0.1/8 ::1 192.168.2.1/24 37.1.83.168 109.195.115.122
```
- Localhost + office + home + vpn
  
Set the IPs you want fail2ban to ignore, the ban time (in seconds) and maximum number of user attempts to your liking: 
```  
  [DEFAULT]
# "ignoreip" can be an IP address, a CIDR mask or a DNS host
ignoreip = 127.0.0.1
bantime  = 3600
maxretry = 3 
```  
Мои значения
bantime  = 36000
```  
# A host is banned if it has generated "maxretry" during the last "findtime"
# seconds.
findtime  = 10m
  
# "maxretry" is the number of failures before a host get banned.
maxretry = 3
```  
И надо рестартовать fail2ban чтобы он заработал
```
sudo /etc/init.d/fail2ban restart 
```  
  From <https://help.ubuntu.com/community/Fail2ban>

Гуй можно настраивать и до fail2ban. Но fail2ban - must have, если у нас "в сеть" торчит 22й порт. В логе авторизации, пока я fail2ban не настроил, простоянно валились сообщения о неправильном юзере и неправильном пароле.

Дополнительные настройки для тюрьмы ssh (файл /etc/fail2ban/jail.local)
```
#
# JAILS
#

#
# SSH servers
#

[sshd]

# To use more aggressive sshd modes set filter parameter "mode" in jail.local:
# normal (default), ddos, extra or aggressive (combines all).
# See "tests/files/logs/sshd" or "filter.d/sshd.conf" for usage example and details.
#mode   = normal
enabled = true
port    = ssh
filter  = sshd
maxretry = 2
logpath = %(sshd_log)s
backend = %(sshd_backend)s
```
не забываем рестартовать сервис после редактирования конфигурации
```
sudo systemctl restart fail2ban
sudo systemctl status fail2ban
```
и ещё можно проверить, что там уже заблокировано
```
fail2ban-client status sshd
```

## Шаг 12 - гуй
Заходим на порт 5000 вашего сервера admin/admin

![заходим в global settings](/images/photo_2022-04-09_23-14-59.jpg)

Серверу надо знать свой внешний ip адрес

Я в конфигурацию вводил полное имя словами, гуй (сыроват) превращает его в ipv4

Теперь генерим ключи сервера

![генерим ключи сервера - Приватник и публик](/images/photo_2022-04-09_23-17-40.jpg)

Обязательно добавляем скрипты PostUp and PostDown
```
Post Up
iptables -t nat -A POSTROUTING -o ens3 -j MASQUERADE; ip6tables -t nat -A POSTROUTING -o ens3 -j MASQUERADE
Post Down
iptables -t nat -D POSTROUTING -o ens3 -j MASQUERADE; ip6tables -t nat -D POSTROUTING -o ens3 -j MASQUERADE
```
Тут внимательно: имя интерфейса ens3 может отличаться. Для того, чтобы его увидеть, надо набрать в ssh коминду
```
ip a
```
Имя здесь:

![check interface name](/images/photo_2022-04-09_23-20-16.jpg)

Создаём клиентов

![здесь же создаём клиентов](/images/photo_2022-04-09_23-20-51.jpg)

Не забываем сохранять.

И тут я налетел несколько раз уже из-за склероза - обязательно надо нажимать кнопку apply configuration после добавления каждого клиента.

![apply](/images/photo_2022-04-09_23-24-43.jpg)

Клиенту на Андроиде и Айфоне можно показать картинку qr-code, и у них всё заработает

![scan](/images/photo_2022-04-09_23-26-12.jpg)


Можно скачать файл конфигурации через download. Этот вариант подходит для компьютеров и роутеров.

Внимание, в файле приватный ключ. Берегите его, не надо просто так везде его хранить, лучше в другой раз скачать заново с сервера.

## Виндовый клиент
https://www.wireguard.com/install/

- в него надо просто "подгрузить" его персональный конфиг и подключаться.

Осторожно. Я "подключил" свой компьютер, когда зашел в него удалённо. Конечно, я его тут же и потерял до тех пор, пока не добрался до него пешком.

Когда клиент нормально законнектился (и на андроиде, и на ios, и на винде) - у него тикают оба счётчика, и отправленных, и принятых байт

![sample](/images/photo_2022-04-09_23-29-51.jpg)

## Keenetik

добавить WG, подгрузить в него CFG файл и настроить приоритет.

![component](/images/photo_2022-04-10_00-04-32.jpg)

конфигурация

![config](/images/photo_2022-04-10_00-05-02.jpg)

Имя Connection и пира кинетик позволяет заполнить самому

Включить не забываем

![turn on](/images/photo_2022-04-10_00-05-47.jpg)

Надо настроить приоритеты

![priority](/images/photo_2022-04-10_00-06-22.jpg)

![priority2](/images/photo_2022-04-10_00-06-51.jpg)

У меня он заработал, когда я ему приоритет первый поставил

![priority3](/images/photo_2022-04-10_00-07-26.jpg)

И ещё вот эту галочку на WG подключении тоже надо кликнуть.

## Что может пойти не так

1) поменять пароль рута или себя и забыть его. Придётся пересоздавать сервер.
2) заблочить файрволом доступ и не смочь войти в результате.
3) у меня "получалось" пытаться войти, не запустив VPN. 
- чтобы проверить, работает ли WG (в ssh)
```
wg show
```
- быстрый старт-стоп (без сервиса)  SSH при этом отваливается! Если не сконфигурён клиент, можно потеряться!
```
wg-quick up wg0
wg-quick down wg0
```
- чтобы проверить статус сервиса
```
systemctl status wg0 wg-quick@wg0
```
4) "хорошая" идея - настраивать VPN на удалённом компьютере, если там никого нет. Вы включаете VPN клиента и всё. Войти в этот компьютер вы не можете. Впрочем, с роутером то же - можно его настроить и потерять сеть. И к вам придут выяснять, куда делся интернет 😊


5) при включении VPN на кинетике с учётом последней галочки (Use for accessing the internet) у вас меняется внешний IP-шник и что-то может отрубиться. Например, клиент кинопоиска "потерял" всю историю.
По-идее тут можно поиграть командами роутинга, но я не разбирался ещё.


## Дополнительная настройка firwall.
ещё одно дополнение, которое я чуть не забыл. Но я не на 100% уверен, что оно нужно. 

- позволить роутинг от вайргардовского к физическому интерфейсу
```
ufw route allow in on wg0 out on ens3
```
- и настройка правил роутинга в ufw
```
vi /etc/ufw/sysctl.conf:
```
раскомментировать строки
```
net/ipv4/ip_forward=1
net/ipv6/conf/default/forwarding=1
net/ipv6/conf/all/forwarding=1
```
и рестартовать.
```
ufw disable
ufw enable
```
почему не уверен: похоже, это же самое делает Post Up правило самого WG Server.

### альтертанивный WireGuard client. 
https://tunsafe.com

В Андроиде у него есть список приложений, которые можно от него отключить (по-умолчанию все включены) (работает, я проверял, исключая FireFox - при этом перестал открываться сайт bbc . com.)

Общая идея - для телефона те приложения, кому надо, идут через ВПН, другие - напрямую. Некоторым клиентам "моего домашнего" впн это очень надо.

Подключение то же самое, что и в родном клиенте - через картинку.

### исключение локальных (и других не-VPN) адресов из списка AllowedIPs

Есть средство для исключения локальных (или тех, которые хочется) адресов из списка wireguard. Неудобно, но работает. Особенно полезно если WG установлен на работе - как раз рабочие адреса можно исключить и тогда если комп залочится, то домен он всё равно будет видеть.

https://www.procustodibus.com/blog/2021/03/wireguard-allowedips-calculator/

Калькулятор формирует список разрешенных масок, исключая тот, что надо оставить разрешенным. 

Для "все на WG разрешенные = 0.0.0.0/0" + "а рабочие оставить вне wg = !разрешенные=10.10.0.0/16" 

получается вот такая строка для AllowedIPs в конфиг пира:
```
AllowedIPs = 10.0.0.0/24, 10.0.2.0/23, 10.0.4.0/22, 10.0.8.0/21, 10.0.16.0/20, 10.0.32.0/19, 10.0.64.0/18, 10.0.128.0/17, 10.1.0.0/16, 10.2.0.0/15, 10.4.0.0/14, 10.8.0.0/13, 10.16.0.0/12, 10.32.0.0/11, 10.64.0.0/10, 10.128.0.0/9
```
Проверил, работает.

Вроде всё. Если есть вопросы - пишите, бум решать.
