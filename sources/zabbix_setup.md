# Настройка Zabbix от начала до конца
### Установка и настройка Ubuntu Server

###### Установка
Устанавливаем систему по-умолчанию. Здесь и далее я произвожу установку с ubuntu-16.04.3-server-amd64.iso на Hyper-V 2012 R2.
При создании виртуальной машины, я выбираю Поколение 2.0, однако Security Boot в Firmware я перед запуском отключаю. Это связано с особенностями Windows Server, который не полностью поддерживает ОС Linux.
Во время установки отключаю автоматические обновления, а также отказываюсь от установки дополнительных пакетов, кроме OpenSSH Server, а все остальное я установлю в ручном режиме после первоначальной настройки системы.

###### Начальная настройка
Первым делом обновим установленные пакеты:
> sudo apt-get update && sudo apt-get upgrade
> sudo reboot

Настраиваем статический IP
> sudo nano /etc/network/interfaces

комментируем строку
> iface eth0 inet dhcp

и вставляем следующие строки
> iface eth0 inet static
> address 192.168.40.10
> netmask 255.255.255.0
> gateway 192.168.40.1
> dns-nameservers 192.168.40.2 192.168.40.3

и перезапускаем сеть
> sudo /etc/init.d/networking restart

###### Установка и настройка LAMP
Сначала поставим Apache2 и необходимые библиотеки
> sudo apt-get install apache2 libapache2-mod-php
> sudo systemctl enable apache2
> sudo systemctl start apache2

Устанавливаем MySQL и библиотеки
> sudo apt-get install mysql-server php7.0-mysql
> sudo systemctl enable mysql
> sudo systemctl start mysql

Теперь ставим PHP7.0 и его модули
> sudo apt-get install php7.0 php7.0-mysql php7.0-curl php7.0-gd php7.0-json php7.0-opcache php7.0-xml mcrypt php7.0-mcrypt

Правим часовой пояс
> root@zabbix:~# sudo nano /etc/php/7.0/apache2/php.ini
[...]
date.timezone = Europe/Moscow
[...]
> sudo systemctl restart apache2

###### Устанавливаем Zabbix
Скачиваем и настраиваем репозиторий zabbix
> wget http://repo.zabbix.com/zabbix/3.4/ubuntu/pool/main/z/zabbix-release/zabbix-release_3.4-1+xenial_all.deb
> sudo dpkg -i zabbix-release_3.4-1+xenial_all.deb
> sudo apt-get update

Теперь ставим необдимые нам пакеты, в частности Server и Fronend для Zabbix, а также его агент Agent и вспомогательные службы для мониторинга самого серевера.
> sudo apt-get install zabbix-server-mysql zabbix-frontend-php zabbix-agent zabbix-get zabbix-sender snmp snmpd snmp-mibs-downloader php7.0-bcmath php7.0-xml php7.0-mbstring

Настроим правильный часовой пояс для Zabbix
> sudo nano /etc/zabbix/apache.conf
> [...]
> php_value date.timezone Europe/Moscow
> [...]

Создадим базу данных в mysql для нашего сервера и настроим нужные привилегии
> mysql -u root -p
> mysql> create database zabbixdb character set utf8 collate utf8_bin;
> mysql> grant all privileges on zabbixdb.* to zabbixuser@localhost identified by 'Password';
> mysql> flush privileges;
> mysql> quit

Импортируем схемы и данные в созданную нами базу данных
> zcat /usr/share/doc/zabbix-server-mysql/create.sql.gz | mysql -u root -p zabbixdb

Правим конфигурационные файлы для сервера и подключаем службы Zabbix
> sudo nano /etc/zabbix/zabbix_server.conf
> [...]
> DBHost=localhost
> DBName=zabbixdb
> DBUser=zabbixuser
> DBPassword=Password
> [...]
> sudo systemctl reload apache2
> sudo systemctl restart apache2
> sudo systemctl enable zabbix-server
> sudo systemctl start zabbix-server
> sudo systemctl enable zabbix-agent
> sudo systemctl start zabbix-agent

###### Завершение настройки Zabbix
Переходим по адресу http://server/zabbix и проходим по всем шагам настройки Мастера, на третьем шаге указываем данные, которые мы записали ранее, в частности Database, User, Password. Database port оставляем по-умолчанию.
На четвертом шаге в поле Name указываем произвольное название нашего сервера
Для входа в админку используем имя Admin и zabbix в качестве пароля

###### Дополнительные настройки Zabbix
> sudo nano /etc/zabbix/zabbix_server.conf
> [...]
> CacheSize=256M
> StartDiscoverers=5
> StartPingers=5
> StartPollers=5
> StartPollersUnreachable=1
> [...]
> sudo systemctl restart zabbix-server

###### Ставим Grafana
> sudo nano /etc/apt/sources.list
> [...]
> deb https://packagecloud.io/grafana/testing/debian/ jessie main
> [...]
> curl https://packagecloud.io/gpg.key | sudo apt-key add -
> sudo apt-get update
> sudo apt-get install grafana
> sudo systemctl daemon-reload
> sudo systemctl enable grafana-server
> sudo systemctl start grafana-server
> sudo systemctl status grafana-server

Ставим плагин для Zabbix
> sudo grafana-cli plugins install alexanderzobnin-zabbix-app
> sudo systemctl restart grafana-server

Настроим нужные привилегии для того чтобы grafana могла напрямую подключаться к базе zabbix
> mysql -u root -p
> mysql> grant select on zabbixdb.* to grafana@localhost identified by 'grafana-zabbix';
> mysql> flush privileges;
> mysql> quit