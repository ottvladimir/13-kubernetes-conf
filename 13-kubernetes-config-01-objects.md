# Домашнее задание к занятию "13.1 контейнеры, поды, deployment, statefulset, services, endpoints"
Настроив кластер, подготовьте приложение к запуску в нём. Приложение стандартное: бекенд, фронтенд, база данных (пример можно найти в папке 13-kubernetes-config).

## Задание 1: подготовить тестовый конфиг для запуска приложения
Для начала следует подготовить запуск приложения в stage окружении с простыми настройками. Требования:
* под содержит в себе 3 контейнера — фронтенд, бекенд, базу;
* регулируется с помощью deployment фронтенд и бекенд;
* база данных — через statefulset.
создаю namespace stage 
```bash
$ kubectl create namespace stage
namespace/stage created
```
Запускаю деплоймент
```bash
$ kubectl apply -f  onefiledeployment.yml -n stage
```
```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: myapp
  name: frontend-backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - image: ottvladimir/frontend:main
        name: frontend
        ports:
        - containerPort: 80
      - image: ottvladimir/backend:main
        name: backend
        ports:
        - containerPort: 9000
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgresql-db
spec:
  serviceName: “postgresql-db”
  selector:
    matchLabels:
      app: postgresql-db
  replicas: 1
  template:
    metadata:
      labels:
        app: postgresql-db
    spec:
      containers:
      - name: postgresql-db
        image: postgres:13-alpine
        ports:
        - containerPort: 5432
        env:
          - name: POSTGRES_DB
            value: news
          - name: POSTGRES_PASSWORD
            value: postgres
          - name: POSTGRES_USER
            value: postgres
---
apiVersion: v1
kind: Service
metadata:
  name: postgresql-db
spec:
  selector:
    app: postgresql-db
  ports:
    - protocol: TCP
      port: 5432
      targetPort: 5432
```
Проверяю
```bash
$ kubectl get po -n stage 
NAME                               READY   STATUS    RESTARTS   AGE
frontend-backend-975d9c94c-t66vc   2/2     Running   0          11m
```
Открываю порт
```bash
$  
Forwarding from 127.0.0.1:80 -> 80
Forwarding from [::1]:80 -> 80
```
Проверяю
```bash
$ curl -l localhost:80 
<!DOCTYPE html>
<html lang="ru">
<head>
    <title>Список</title>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <link href="/build/main.css" rel="stylesheet">
</head>
<body>
    <main class="b-page">
        <h1 class="b-page__title">Список</h1>
        <div class="b-page__content b-items js-list"></div>
    </main>
    <script src="/build/main.js"></script>
</body>
```
## Задание 2: подготовить конфиг для production окружения
Следующим шагом будет запуск приложения в production окружении. Требования сложнее:
* каждый компонент (база, бекенд, фронтенд) запускаются в своем поде, регулируются отдельными deployment’ами;
* для связи используются service (у каждого компонента свой);
* в окружении фронта прописан адрес сервиса бекенда;
* в окружении бекенда прописан адрес сервиса базы данных.

## Задание 3 (*): добавить endpoint на внешний ресурс api
Приложению потребовалось внешнее api, и для его использования лучше добавить endpoint в кластер, направленный на это api. Требования:
* добавлен endpoint до внешнего api (например, геокодер).

---

### Как оформить ДЗ?

Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.

В качестве решения прикрепите к ДЗ конфиг файлы для деплоя. Прикрепите скриншоты вывода команды kubectl со списком запущенных объектов каждого типа (pods, deployments, statefulset, service) или скриншот из самого Kubernetes, что сервисы подняты и работают.

---
