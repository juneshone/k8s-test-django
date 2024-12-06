# Запуск веб-приложения в облачном кластере Kubernetes в Яндекс Облаке

## Как подготовить окружение

Установите и инициализируйте интерфейс командной строки [Yandex Cloud](https://yandex.cloud/ru/docs/cli/quickstart#install).

Добавьте учетные данные кластера Kubernetes в конфигурационный файл kubectl:

```
yc managed-kubernetes cluster get-credentials --id ID --external
```

Используйте утилиту kubectl для работы с кластером Kubernetes:

```
kubectl get pods
```

Далеее в командах используется namespace кластера K8s `edu-frosty-dewdney`. Замените на свой.

Чтобы секреты не утекли случайно в сеть соберите их в специальный отдельный манифест `django-secret.yaml` и поместите в него переменные окружения. Храните его отдельно от репозитория. Добавьте внутрь кластера новый объект Secret командой:

```
kubectl apply -f .\deploy\yc-sirius\edu-frosty-dewdney\dev\django-secret.yaml --namespace=edu-frosty-dewdney
```

Запустите команду, которая отображает подробный список секретов:

```
kubectl get secrets
```

На отдельном сервере снаружи кластера k8s работает СУБД Managed PostgreSQL. Она защищена SSL-сертификатом, который находится в секрете `pg-root-cert`. Доступы к базе данных PostgreSQL должны лежать в секрете K8s `postgres`.

Если нет секрета `pg-root-cert` с SSL-сертификатом для подключения к вашей базе данных PostgreSQL, то получите сертификат согласно [инструкции](https://yandex.cloud/ru/docs/managed-postgresql/operations/connect), установите значение в `.\deploy\yc-sirius\edu-frosty-dewdney\dev\psql-secret.yaml`и выполните команду:

```
kubectl apply -f .\deploy\yc-sirius\edu-frosty-dewdney\dev\psql-secret.yaml --namespace=edu-frosty-dewdney
```

При развертывании приложения SSL-сертификат монтируется как файл внутри контейнера, делая его доступными через файловую систему.

## Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).

## Как запустить веб-приложение в dev окружении

СУБД уже запущена отдельно от кластера Kubernetes, снаружи.
Для экспериментов с psql в кластере k8s разверните веб-приложение:

```
kubectl apply -f .\deploy\yc-sirius\edu-frosty-dewdney\dev\django-deployment.yaml --namespace=edu-frosty-dewdney
kubectl apply -f .\deploy\yc-sirius\edu-frosty-dewdney\dev\django-service.yaml --namespace=edu-frosty-dewdney
```

Убедитесь, что развертывание находится в состоянии готовности, а служба создана и доступна на узловом порту:

```
kubectl get all
```

Примените миграции:

```shell
kubectl apply -f .\deploy\yc-sirius\edu-frosty-dewdney\dev\django-job-migrate.yaml --namespace=edu-frosty-dewdney
```

Войдите в под с django и создайте суперпользователя

```shell
kubectl exec -i -t -n NAMESPACE POD_NAME -c CONTAINER_NAME -- sh -c "clear; (bash || ash || sh)"
python manage.py createsuperuser
```

Готово. Сайт будет доступен по адресу [https://edu-frosty-dewdney.sirius-k8s.dvmn.org](https://edu-frosty-dewdney.sirius-k8s.dvmn.org). Вход в админку находится по адресу [https://edu-frosty-dewdney.sirius-k8s.dvmn.org/admin/](https://edu-frosty-dewdney.sirius-k8s.dvmn.org/admin/).

## Настройка регулярного задания

1. Запустите регулярную задачу на удаление сессий с помощью манифестфайла:

```shell
kubectl apply -f .\deploy\yc-sirius\edu-frosty-dewdney\dev\django-clearsessions.yaml --namespace=edu-frosty-dewdney
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
kubectl apply -f .\deploy\yc-sirius\edu-frosty-dewdney\dev\django-job-migrate.yaml --namespace=edu-frosty-dewdney
```

_Пример успешного запуска команды_:

```sh
$ kubectl logs job-migrate-xcss4   
Operations to perform:
  Apply all migrations: admin, auth, contenttypes, sessions
Running migrations:
  No migrations to apply.
```

## Как выгрузить образ на Docker Hub

В терминале, находясь в директории с вашим Dockerfile, выполните команду для создания образа:

```bash
docker build -t my-django-app -f Dockerfile .
```

Войдите в свою учетную запись Docker Hub через терминал:

```bash
docker login 
```

Тегируйте ваш образ, чтобы он соответствовал вашему репозиторию на Docker Hub. Например:

```bash
docker tag my-django-app your_dockerhub_username/my-django-app:latest
```

Теперь вы можете выгрузить свой образ на Docker Hub:

```bash
docker push your_dockerhub_username/my-django-app:latest
```