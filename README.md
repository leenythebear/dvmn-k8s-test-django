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


## Как запустить проект в кластере Minikube

1. Запустите кластер Minikube командой:
```
minikube start
```
2. Сбилдите и загрузите образ приложения Django в Minikube командой (находясь в папке с Dockerfile):
```
minikube image build . -t django:1
```
3. Установите helm: [инструкции по установке для различных ОС](https://helm.sh/docs/intro/install/)

4. Установите Helm Chart for PostgreSQL:
```
helm repo add stable https://charts.helm.sh/stable
helm repo update
helm install my-db oci://registry-1.docker.io/bitnamicharts/postgresql
```
5. При необходимости подключитесь к базе данных:
```
 kubectl run my-db-postgresql-client --rm --tty -i --restart='Never' --namespace default --image docker.io/bitnami/postgresql:15.3.0-debian-11-r24 --env="PGPASSWORD=$POSTGRES_PASSWORD" \
      --command -- psql --host my-db-postgresql -U postgres -d postgres -p 5432
```
6. Создайте superuser:
```
kubectl exec -it <POD_NAME> --container <NAME> -- bash
./manage.py createsuperuser
```
7. Создайте файл `configmap.yaml` и внесите в него переменные по примеру:
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: django-config
data:
  SECRET_KEY: ...
  DATABASE_URL: ...
  DEBUG: ...
  ALLOWED_HOSTS: star-burger.test
```
8. Пропишите в файле /etc/hosts на своём компьютере домен star-burger.test
9. Запустите миграции командой:
```
kubectl apply -f migrate.yaml
```
10. Запустите Ingress командой:
```
kubectl apply -f ingress.yaml
```
11. Запустите проект командой:
```
kubectl apply -f django.yaml && kubectl apply -f configmap.yaml && kubectl apply -f django-clearsessions.yaml 
```
12. Перейдите по адресу [http://star-burger.test/](http://star-burger.test/)