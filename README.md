# Django Site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри контейнера Django приложение запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

## Как подготовить окружение к локальной разработке

Код в репозитории полностью докеризирован, поэтому для запуска приложения вам понадобится Docker. Инструкции по его установке ищите на официальных сайтах:

- [Get Started with Docker](https://www.docker.com/get-started/)

Вместе со свежей версией Docker к вам на компьютер автоматически будет установлен Docker Compose. Дальнейшие инструкции будут его активно использовать.

## Как запустить сайт для локальной разработки

Запустите базу данных и сайт:

```shell
$ docker compose up
```

В новом терминале, не выключая сайт, запустите несколько команд:

```shell
$ docker compose run --rm web ./manage.py migrate  # создаём/обновляем таблицы в БД
$ docker compose run --rm web ./manage.py createsuperuser  # создаём в БД учётку суперпользователя
```

Готово. Сайт будет доступен по адресу [http://127.0.0.1:8080](http://127.0.0.1:8080). Вход в админку находится по адресу [http://127.0.0.1:8000/admin/](http://127.0.0.1:8000/admin/).

## Как вести разработку

Все файлы с кодом django смонтированы внутрь докер-контейнера, чтобы Nginx Unit сразу видел изменения в коде и не требовал постоянно пересборки докер-образа -- достаточно перезапустить сервисы Docker Compose.

### Как обновить приложение из основного репозитория

Чтобы обновить приложение до последней версии подтяните код из центрального окружения и пересоберите докер-образы:

``` shell
$ git pull
$ docker compose build
```

После обновлении кода из репозитория стоит также обновить и схему БД. Вместе с коммитом могли прилететь новые миграции схемы БД, и без них код не запустится.

Чтобы не гадать заведётся код или нет — запускайте при каждом обновлении команду `migrate`. Если найдутся свежие миграции, то команда их применит:

```shell
$ docker compose run --rm web ./manage.py migrate
…
Running migrations:
  No migrations to apply.
```

### Как добавить библиотеку в зависимости

В качестве менеджера пакетов для образа с Django используется pip с файлом requirements.txt. Для установки новой библиотеки достаточно прописать её в файл requirements.txt и запустить сборку докер-образа:

```sh
$ docker compose build web
```

Аналогичным образом можно удалять библиотеки из зависимостей.

<a name="env-variables"></a>
## Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).

## Создание Secrets Kubernetes в кластере

Чтобы секреты не утекли случайно в сеть соберите их в специальный отдельный манифест secret.yaml и поместите в него переменные окружения. Храните его отдельно от репозитория. 

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: secrets
type: Opaque
stringData:
  secret_key: SECRET_KEY
  database_url: DATABASE_URL
  allowed_hosts: ALLOWED_HOSTS
  debug: DEBUG
```

Добавьте внутрь кластера новый объект Secret командой:

```
kubectl apply -f .\local-minikube-virtualbox\secret.yaml
```


## Настройка Ingress

Включите контроллер входа NGINX, выполнив следующую команду:

```
minikube addons enable ingress
```

Убедитесь, что контроллер входа NGINX запущен:

```
kubectl get pods -n ingress-nginx
```

Разверните приложение:

```
kubectl apply -f .\local-minikube-virtualbox\deployment.yaml
kubectl apply -f .\local-minikube-virtualbox\service.yaml
```

Убедитесь, что развертывание находится в состоянии готовности, а служба создана и доступна на узловом порту:

```
kubectl get all
```

Создайте объект Ingress, выполнив следующую команду:

```
kubectl apply -f .\local-minikube-virtualbox\ingress.yaml
```

Убедитесь, что IP-адрес установлен:

```
kubectl get ingress
```

Найдите внешний IP-адрес, указанный minikube и добавьте строку в конец файла /etc/hosts на вашем компьютере (вам потребуется доступ администратора). Инструкцию смотреть [здесь](https://help.reg.ru/support/dns-servery-i-nastroyka-zony/rabota-s-dns-serverami/fayl-hosts-gde-nakhoditsya-i-kak-yego-izmenit#2).:

```
minikube ip
```

_Пример_:

```
172.17.0.15 hello-world.example
```


## Настройка регулярного задания

1. Запустите регулярную задачу на удаление сессий с помощью манифестфайла:

```shell
kubectl apply -f .\local-minikube-virtualbox\django-clearsessions.yaml
```

Для запуска задание вручную в целях тестирования:

```shell
kubectl create job --from=cronjob/<cronjob-name> <job-name> -n <namespace-name>
```

_Пример успешной работы_:
```shell
$ kubectl get jobs --watch
NAME                            STATUS     COMPLETIONS   DURATION   AGE
django-clearsessions-28854275   Complete   1/1           7s         2m21s
django-clearsessions-28854276   Complete   1/1           7s         81s
```

2. Запустите задачу на применение миграций с помощью манифестфайла:

```shell
kubectl apply -f .\local-minikube-virtualbox\job-migrate.yaml
```

_Пример успешного запуска команды_:

```sh
$ kubectl logs job-migrate-xcss4   
Operations to perform:
  Apply all migrations: admin, auth, contenttypes, sessions
Running migrations:
  No migrations to apply.

```


## Создание базы данных postgresql внутри кластера

1. Установите [Helm](https://helm.sh/) и выполните команду:

```shell
helm install django-postgresql --set auth.username=PG_USERNAME --set auth.password=PG_PASSWORD --set auth.database=PG_DATABASE --set service.ports.postgresql=PG_PORT oci://registry-1.docker.io/bitnamicharts/postgresql
```

где `PG_USERNAME` - пользователь бд, `PG_PASSWORD` - пароль бд, `PG_DATABASE` - название бд, `PG_PORT` - порт бд

Для подключения к вашей базе данных выполните следующую команду:

```shell
kubectl run djangoapp-postgresql-client --rm --tty -i --restart='Never' --namespace default --image docker.io/bitnami/postgresql:17.0.0-debian-12-r11 --env="PGPASSWORD=PG_PASSWORD" --command -- psql --host djangoapp-postgresql -U PG_USERNAME -d PG_DATABASE -p PG_PORT
```


2. Примените миграции:

```shell
kubectl apply -f .\local-minikube-virtualbox\job-migrate.yaml
```

3. Войдите в под с django и создайте суперпользователя

```shell
kubectl exec -it POD bash
python manage.py createsuperuser
```

3. Поместите урл `postgres://PG_USERNAME:PG_PASSWORD@django-postgresql:PG_PORT/PG_DATABASE` для подключения к базе данных в манифест с секретом.