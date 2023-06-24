# Django site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри конейнера Django запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

## Как запустить dev-версию

Запустите базу данных и сайт:

```shell-session
$ docker-compose up
```

В новом терминале не выключая сайт запустите команды для настройки базы данных:

```shell-session
$ docker-compose run web ./manage.py migrate  # создаём/обновляем таблицы в БД
$ docker-compose run web ./manage.py createsuperuser
```

Для тонкой настройки Docker Compose используйте переменные окружения. Их названия отличаются от тех, что задаёт docker-образа, сделано это чтобы избежать конфликта имён. Внутри docker-compose.yaml настраиваются сразу несколько образов, у каждого свои переменные окружения, и поэтому их названия могут случайно пересечься. Чтобы не было конфликтов к названиям переменных окружения добавлены префиксы по названию сервиса. Список доступных переменных можно найти внутри файла [`docker-compose.yml`](./docker-compose.yml).

## Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).

## Minikube
1. Установите [Minikube](https://kubernetes.io/ru/docs/tasks/tools/install-minikube/).
2. Перейти в дирикторию проекта. Собрать образ проекта командой:
```
minikube image build django-image .\backend_main_django
```
3. Разверните PostgreSQL в кластере ([пример](https://artifacthub.io/packages/helm/bitnami/postgresql)). Создайте базу данных, пользователя ([пример](https://medium.com/coding-blocks/creating-user-database-and-adding-access-on-postgresql-8bfcd2f4a91e)).
4. Создать `config.yaml` файл со следующим содержимым:
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: dj-conf
data:
  DATABASE_URL: указать необходимое
  SECRET_KEY: указать необходимое
  DEBUG: "False"
  ALLOWED_HOSTS: "star-burger.test"
```
Далее, выполнить команду в вашем кластере:
```
kubectl apply -f kubernetes\config.yaml
```
5. Задеплойте сайт на Джанго:
```
kubectl apply -f kubernetes\deploy.yaml
```
6. Выполнить миграции командой:
```
kubectl apply -f kubernetes\migrate-job.yaml
```
7. Пропишите на своей машине в /etc/hosts (для linux) или в C:\Windows\System32\drivers\etc\hosts домен star-burger.test, сопоставить с IP виртуальной машины (узнать ip c помощью команды minikube ip).
Далее запустите команду:
```
kubectl apply -f kubernetes\ingress.yaml
```
8. Можно посмотреть сайт по адресу [http://star-burger.test/](http://star-burger.test/).
