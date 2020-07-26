# Домашнее задание №2

### Команда установки БД из helm, вместе с файлом values.yaml

helm install mngdb bitnami/mongodb -f values.yaml

### Команда применения первоначальных миграций

kubectl apply -f job-init.yaml

### Команда kubectl apply -f, которая запускает в правильном порядке манифесты кубернетеса

kubectl apply -f config.yaml -f deployment.yaml -f service.yaml -f ingress.yaml

### Postman-коллекция, в которой представлены примеры запросов к сервису.

_Базовый url в коллекции – arch.homework_

newman run .\Otus_Arch_ilyakozyrev.postman_collection.json

### Задание со звёздочкой

helm install kia-helm-chart .\kia-helm-chart
