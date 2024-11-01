# OM-DZ02
На виртуальной машине установлена CMS wordpress, которая включает в себя следующие компоненты: 

nginx, php-fpm, database (MySQL)

На CMS развернут тестовый домен http://www.example.com/

---

Скачиваем архив бинарника последней версии VictoriaMetrics
 
wget https://github.com/VictoriaMetrics/VictoriaMetrics/releases/download/v1.105.0/victoria-metrics-linux-amd64-v1.105.0.tar.gz

Распакуем скачанный архив:

tar zxf victoria-metrics-linux-amd64-v1.105.0.tar.gz -C /usr/local/bin/

Создаем служебного пользователя victoriametrics:

useradd -r -c 'VictoriaMetrics TSDB Service' victoriametrics

Также создаем каталоги для хранения данных и идентификатора процесса:

mkdir -p /var/lib/victoriametrics /run/victoriametrics

Зададим в качестве владельца для созданных каталогов нашего нового пользователя:

chown victoriametrics:victoriametrics /var/lib/victoriametrics /run/victoriametrics

Теперь создадим файл юнита:

vi /etc/systemd/system/victoriametrics.service

---

[Unit]

Description=VictoriaMetrics

After=network.target


[Service]

Type=simple

User=victoriametrics

PIDFile=/run/victoriametrics/victoriametrics.pid

ExecStart=/usr/local/bin/victoria-metrics-prod -storageDataPath /var/lib/victoriametrics -retentionPeriod 14d

ExecStop=/bin/kill -s SIGTERM $MAINPID

StartLimitBurst=5

StartLimitInterval=0

Restart=on-failure

RestartSec=1


[Install]

WantedBy=multi-user.target

---

Перечитываем конфигурацию systemd:

systemctl daemon-reload

Разрешаем запуск юнита и стартуем его:

systemctl enable victoriametrics --now

Проверим состояние сервиса:

systemctl status victoriametrics

По умолчанию VictoriaMetrics слушает на порту 8428. Мы можем сделать запрос с помощью curl:

curl 127.0.0.1:8428

---

Хранение метрик Prometheus в VictoriaMetrics

Прежде чем начать настройку, с сервера Prometheus проверим доступность VictoriaMetrics:

curl 192.168.56.103:8428

Скорректируем конфигурационный файл Prometheus prometheus.yml

Добавляем в global:

  external_labels:
    site: prod

Опция external_labels позволяет навесить тег, 

с помощью которого мы будем понимать, что конкретные метрики пришли 

с конкретного сервера Prometheus.

И добавляем опцию:

remote_write:

  - url: http://192.168.56.103:8428/api/v1/write

      queue_config:

        max_samples_per_send: 10000

        capacity: 20000

        max_shards: 30
	  
	  
Запускаем сервер prometheus

docker-compose up -d


В веб-интерфейс VictoriaMetrics	можно посмотреть метрики:

http://192.168.56.103:8428/vmui/


