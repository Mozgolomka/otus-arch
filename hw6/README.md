# Домашнее задание №6
# Идемпотетность и коммутативность API в HTTP и очередях.

### Схема взаимодействия сервисов.

![hw6](arch_hw6.png)

Реализация идемпотентности для создания заказа с помощью паттерна Transactional outbox:
1) В запросе передаём новый заголовок X-Request-Id с идентификатором запроса;
2) В ходе обработки запроса первоначально создаём запись в таблице IdempotentAction с уникальным полем по идентификатору запроса;
3) Если на шаге 2 база вернула ошибку дублирования значений, тогда забираем данные, полученные при выполнении запроса с таким идентификатором в прошлый раз, и отправляем их пользователю. Запрос обработан.
4) Если шаг 2 прошёл без проблем, значит пришёл уникальный запрос. Создаём запись в таблице заказ со статусом "new".
5) Возвращаем идентификатор заказа пользователю. Запрос обработан.
Затем за работу принимается шедулер:
1) Сначала он ищет в таблице IdempotentAction все необработанные запросы.
2) Затем для каждого запроса по заказу отправляет сообщение в Kafka в рамках топика "order-created".
3) И в конце концов происходит обновление статуса заказа с "new" на "processing".
4) А в самом конце ещё запускается чистка старых (созданных ранее вчерашнего дня) записей в таблице IdempotentAction.

### Инструкция по установке и запуску сервисов.

Приложение будет располагаться в трёх namespace: auth, app и broker. Создаём их.

```
➜  kubectl create namespace auth
namespace/auth created
➜  kubectl create namespace app
namespace/app created
➜  kubectl create namespace broker
namespace/broker created
```

Переключаемся в пространство broker и ставим там сборник Kafka от confluent при помощи helm.

```bash
➜  helm repo add confluentinc https://confluentinc.github.io/cp-helm-charts/
➜  helm repo update
➜  kubectl config set-context --current --namespace=broker
➜  helm install cp confluentinc/cp-helm-charts -f cp_values.yaml
```

Собираем и запускаем сервис работы с пользователями в namespace/auth. Этот сервис содержит в себе CRUD для пользователей, доступный только администраторам, а также функционал по регистрации, восстановлению пароля, аутентификации и авторизации.

```bash
➜  kubectl config set-context --current --namespace=auth
➜  helm install mngdb-auth bitnami/mongodb -f mngdb-auth-values.yaml
➜  cd user
➜  kubectl apply -f .
```

В namespace/app будет обитать сервис заказов order. Для него нужна отдельная база.
```bash
➜  kubectl config set-context --current --namespace=app
➜  cd ../
➜  helm install mngdb bitnami/mongodb -f mngdb-values.yaml
➜  cd order
➜  kubectl apply -f .
```

### Тестирование работоспособности функционала.

Запускаем тесты с помощью newman.

```bash
➜  cd ../
➜  newman run otusArch_KIA_hw6.postman_collection.json
```

И вот что получаем:

```bash
Otus Arch hw6

→ Sign Up User
  POST arch.homework/signup [201 Created, 919B, 350ms]
  √  [INFO] Request: {
    "username": "Vinnie66",
    "firstName": "Hillard",
    "lastName": "Kemmer",
    "phone": "+79998887766",
    "email": "curt_lowe47@yahoo.com",
    "password": "05wfPLgeYAMbH5M",
    "passwordConfirm": "05wfPLgeYAMbH5M"
}
  √  [INFO] Response: {"status":"success","token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjVmYjZkNzJhYjk2MTdiYjU2ODFlYTg4YiIsImlhdCI6MTYwNTgxODE1NCwiZXhwIjoxNjEzNTk0MTU0fQ.uBWLRqAXLBqgKXE8THDmPQK_Ceo8eNl0MM1dmONCi74","data":{"user":{"photo":"default.jpg","role":"player","active":true,"_id":"5fb6d72ab9617bb5681ea88b","username":"Vinnie66","firstName":"Hillard","lastName":"Kemmer","email":"curt_lowe47@yahoo.com","phone":"+79998887766","__v":0}}}

→ Make Order
  POST arch.homework/api/v1/order/ [201 Created, 328B, 31ms]
  √  [INFO] Request: {
    "price": 400
}
  √  [INFO] Response: {"status":"OK","message":"Your order is on the way","orderId":"5fb6d72cb0e9cb593e2c0be8"}

→ Make Order Copy
  POST arch.homework/api/v1/order/ [201 Created, 328B, 11ms]
  √  [INFO] Request: {
    "price": 400
}
  √  [INFO] Response: {"status":"OK","message":"Your order is on the way","orderId":"5fb6d72cb0e9cb593e2c0be8"}

→ Get All Orders
  GET arch.homework/api/v1/order/ [200 OK, 2.91KB, 20ms]
  √  [INFO] Request: [object Object]
  √  [INFO] Response: {"status":"success","results":1,"data":{"data":[{"price":400,"date":"2020-11-19T18:51:12.199Z","_id":"5fb6d72cb0e9cb593e2c0be8","userId":"5fb6d72ab9617bb5681ea88b","status":"new"}]}}

→ Make New Order
  POST arch.homework/api/v1/order/ [201 Created, 328B, 11ms]
  √  [INFO] Request: {
    "price": 400
}
  √  [INFO] Response: {"status":"OK","message":"Your order is on the way","orderId":"5fb6d73317949b6b0459dcc2"}

→ Make New Order Copy
  POST arch.homework/api/v1/order/ [201 Created, 328B, 9ms]
  √  [INFO] Request: {
    "price": 400
}
  √  [INFO] Response: {"status":"OK","message":"Your order is on the way","orderId":"5fb6d73317949b6b0459dcc2"}

→ Get All Orders 2
  GET arch.homework/api/v1/order/ [200 OK, 3.04KB, 13ms]
  √  [INFO] Request: [object Object]
  √  [INFO] Response: {"status":"success","results":2,"data":{"data":[{"price":400,"date":"2020-11-19T18:51:12.199Z","_id":"5fb6d72cb0e9cb593e2c0be8","userId":"5fb6d72ab9617bb5681ea88b","status":"processing"},{"price":400,"date":"2020-11-19T18:51:13.927Z","_id":"5fb6d73317949b6b0459dcc2","userId":"5fb6d72ab9617bb5681ea88b","status":"new"}]}}

┌─────────────────────────┬───────────────────┬───────────────────┐
│                         │          executed │            failed │
├─────────────────────────┼───────────────────┼───────────────────┤
│              iterations │                 1 │                 0 │
├─────────────────────────┼───────────────────┼───────────────────┤
│                requests │                 7 │                 0 │
├─────────────────────────┼───────────────────┼───────────────────┤
│            test-scripts │                12 │                 0 │
├─────────────────────────┼───────────────────┼───────────────────┤
│      prerequest-scripts │                14 │                 0 │
├─────────────────────────┼───────────────────┼───────────────────┤
│              assertions │                14 │                 0 │
├─────────────────────────┴───────────────────┴───────────────────┤
│ total run duration: 16.1s                                       │
├─────────────────────────────────────────────────────────────────┤
│ total data received: 6.16KB (approx)                            │
├─────────────────────────────────────────────────────────────────┤
│ average response time: 63ms [min: 9ms, max: 350ms, s.d.: 117ms] │
└─────────────────────────────────────────────────────────────────┘
```
