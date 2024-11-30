## Как запустить тестовое веб-приложение в dev окружении

Установите и инициализируйте интерфейс командной строки [Yandex Cloud](https://yandex.cloud/ru/docs/cli/quickstart#install).

Добавьте учетные данные кластера Kubernetes в конфигурационный файл kubectl:

```
yc managed-kubernetes cluster get-credentials --id ID --external
```

Используйте утилиту kubectl для работы с кластером Kubernetes:

```
kubectl get pods
```

Разверните веб-приложение:

```
kubectl apply -f .\deploy\yc-sirius\edu-frosty-dewdney\dev\deployment.yaml --namespace=edu-frosty-dewdney
kubectl apply -f .\deploy\yc-sirius\edu-frosty-dewdney\dev\service.yaml --namespace=edu-frosty-dewdney
```

Убедитесь, что развертывание находится в состоянии готовности, а служба создана и доступна на узловом порту:

```
kubectl get all
```

### Как подключиться к защищенному PostgreSQL 

Запустите команду, которая отображает подробный список секретов:

```
kubectl get secrets
```

Если нет секрета `pg-ssl-cert` с SSL-сертификатом для подключения к вашей базе данных PostgreSQL, то получите сертификат согласно [инструкции](https://yandex.cloud/ru/docs/managed-postgresql/operations/connect), установите значение в `.\deploy\yc-sirius\edu-frosty-dewdney\dev\secret.yaml`и выполните команду:

```
kubectl apply -f .\deploy\yc-sirius\edu-frosty-dewdney\dev\secret.yaml --namespace=edu-frosty-dewdney
```

Если есть `pg-ssl-cert` секрет с SSL-сертификатом для подключения к вашей базе данных PostgreSQL, выполните команду:

```
kubectl apply -f .\deploy\yc-sirius\edu-frosty-dewdney\dev\ubuntu-deployment.yaml --namespace=edu-frosty-dewdney
```

Подключитесь к запущенному контейнеру:

```
kubectl exec -i -t -n NAMESPACE POD_NAME -c CONTAINER_NAME -- sh -c "clear; (bash || ash || sh)"
```

Подключитесь к базе данных:

```
psql "host=HOST port=PORT sslmode=require dbname=DBNAME user=USER password=PASSWORD"
```

Для проверки успешности подключения выполните запрос:

```
SELECT version();
```