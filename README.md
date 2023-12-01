# bingo-devops
В рамках тренировок DevOps от Yandex была задача развернуть продуктовую инсталяцию [приложения](https://storage.yandexcloud.net/final-homework/bingo) в соответствии с [ТЗ](https://disk.yandex.ru/i/lHRYD7kWCnQTxg)

Схема стенда, развернутая в облаке Yandex

![image](https://github.com/han-on-duty/bingo-devops/assets/83783443/3289e5f3-83c2-47d4-8b19-8ed51fe8012f)

Развернуть её помог Terraform. main.tf представлен в папке /terraform


**Настройка приложения** 

1. Скачиваем приложение и делаем его исполняемым

`wget https://storage.yandexcloud.net/final-homework/bingo && sudo chmod +x bingo`

2. Команда `./bingo --help` выведет информацию о командах доступных пользователю

3. `./bingo print_current_config` выдаёт ошибку, что config файл не найден, создадим его по шаблону из print_default_config, исправив ip-адрес и логин/пароль для доступа к БД и разместим по пути /opt/bingo/config.yaml

4. К тому же при запуске сервера выпадает ошибка об отсутствии log файла, создадим его `sudo mkdir -p /opt/bongo/logs/c926dff6b3 && sudo ln -s /dev/null  /opt/bongo/logs/c926dff6b3/main.log && sudo chown -h admin:admin /opt/bongo/logs/c926dff6b3/main.log ` здесь я не стал ротацию логов проводить, а сделал ссылку на /dev/null, при необходимости следует сделать по другому

5. Ещё один баг приложения был обнаружен при попытке узнать причину долгого запуска приложения (30с), командой netstat было выявлено, что приложение обращалось по адресу http://google.dns, для исправления протокола я добавил в задании cron строку `@reboot sudo bash -c 'echo "8.8.8.8 google.dns" >> /etc/hosts' && sudo iptables -t nat -A OUTPUT -p tcp --dport 80 -d 8.8.8.8 -j DNAT --to-destination 8.8.8.8:443` которая подменяет порт назначения при обращении к dns серверам google

6. Так же в приложении присутствует rpsLimiter который кладёт приложение через некоторое время поэтому создан healthcheck который проверяет статус ответа на http://localhost:19590/ping, приложение и healthcheker запускаются как systemd юниты. Для того чтобы запустить приложение необходимо скопировать bingo.service и bingo-health-check в директорию /etc/systemd/system/ и выполнить `sudo systemct daemon-reload && sudo systemctl enable bingo.service && sudo systemctl start bingo.service` и аналогичные действия для bingo-health-check. Но перед запуском необходимо настроить БД.

**Настройка БД** 

1. Проводим инсталяцию postgresql и pgbouncer

```
sudo sh -c 'echo "deb https://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'  
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -  
sudo apt update && sudo apt install postgresql pgbouncer -y
```
2. Изменим пароль для доступа в бд и создадим конфигурационный файл userlist.txt для pgbouncer

```
sudo -u postgres psql -c "ALTER USER postgres WITH PASSWORD 'postgres';"
echo "\"postgres\" \"`sudo -u postgres psql -c "select passwd from pg_shadow where usename='postgres';"| grep SCRAM-SHA| cut -c2-`\"" | sudo tee /etc/pgbouncer/userlist.txt
```

3. Укажем параметры для подключения к pgbouncer в /etc/pgbouncer/pgbouncer.ini изменив параметры, указанные ниже

```
[databases]
postgres = host=localhost dbname=postgres
[pgbouncer]
listen_addr = your_ip_address
listen_port = 6432
```
4. Перезагрузим службу `sudo systemctl restart pgbouncer` и на ноде с приложением запустим `./bingo prepare_db` если всё правильно настроить БД начнет наполняться данными

5. Далее создадим индексы для ускорения запросов

```
sudo -u postgres psql -c "create index icustomer on customers (id);"
sudo -u postgres psql -c "create index imovie on movies (id);"
sudo -u postgres psql -c "create index isession on sessions (id);"

```

**Настройка прокси сервера**

1. Устанавливаем nginx `sudo apt update && sudo apt install nginx -y`
2. Соездаем директорию для храненияя кеша `sudo mkdir -p /lib/cache/nginx/cache`
3. Изменяем ip-адреса нод приложений и копируем nginx.conf в директорию /etc/nginx/
4. Чтобы создать самоподписанный сертификат и ключ, запустите команду: `sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/nginx-selfsigned.key -out /etc/ssl/certs/nginx-selfsigned.crt`.Заполните появившиеся поля данными о сервере, которые будут отображаться в сертификате.Самой важной строкой является Common Name (введите FQDN или свое имя). Как правило, в эту строку вносят доменное имя, с которым нужно связать сервер. В случае если доменного имени нет, внесите в эту строку IP-адрес сервера.
5. При использовании OpenSSL нужно также создать ключи Диффи-Хеллмана  `sudo openssl dhparam -out /etc/nginx/dhparam.pem 4096`
6. Создайте новый сниппет Nginx в каталоге /etc/nginx/snippets  `sudo cp self-signed.conf /etc/nginx/snippets/self-signed.conf` 
7. Теперь нужно создать другой сниппет, предназначенный для настроек SSL  `sudo cp ssl-params.conf /etc/nginx/snippets/ssl-params.conf`
8. Перезапустите прокси-сервер `sudo systemctl restart nginx`

Так же важным моментом хорошо построенно инфраструктуры является возможность мониторинга, в yandex мониторинг настроил графики распределения нагрузки на ноды 
![image](https://github.com/han-on-duty/bingo-devops/assets/83783443/4f8e16f0-c3ee-46fb-afe1-1d348de91fe6)


