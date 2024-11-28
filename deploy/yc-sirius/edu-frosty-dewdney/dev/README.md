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
