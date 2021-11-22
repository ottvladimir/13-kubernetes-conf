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
## Задание 2: подготовить конфиг для production окружения
fronend.yml
```yml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: myapp-front
  name: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp-front
  template:
    metadata:
      labels:
        app: myapp-front
    spec:
      containers:
        - image: ottvladimir/frontend:main
          name: frontend
          ports:
            - containerPort: 8000
          env:
            - name: BASE_URL
              value: http://backend:9000
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  selector:
    app: myapp-front
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8000
```
backend.yml
```yml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: myapp-back
  name: backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp-back
  template:
    metadata:
      labels:
        app: myapp-back
    spec:
      containers:
      - image: ottvladimir/backend:main
        name: backend
        ports: 
          - containerPort: 9000
        env:
        - name: DATABASE_URL
          value: postgres://postgres:postgres@db:5432/news
---
apiVersion: v1
kind: Service
metadata:
    name: backend
spec:
  selector:
    app: myapp-back
  ports:
    - protocol: TCP
      port: 9000
      targetPort: 9000
```
postgresql.yml
```yml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgresql
spec:
  selector:
    matchLabels:
      app: myapp-db
  replicas: 1
  template:
    metadata:
      labels:
        app: myapp-db
    spec:
      containers:
      - name: postgresql
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
  name: postgresql
spec:
  selector:
    app: myapp-db
  ports:
    - protocol: TCP
      port: 5432
      targetPort: 5432
```
Стартую
```bash
 kubectl apply -f prod/.                                                                                                                                                                                     
deployment.apps/backend created                                                                                                                                                                                                                                                 
service/backend created                                                                                                                                                                                                                                                         
deployment.apps/frontend created                                                                                                                                                                                                                                                
service/frontend created                                                                                                                                                                                                                                                          
deployment.apps/postgresql created                                                                                                                                                                                                                                              
service/postgresql created 
```
```bash
$ kubectl get all
NAME                             READY   STATUS    RESTARTS   AGE
pod/backend-6599ccf48b-ck9lg     1/1     Running   0          6m20s
pod/frontend-668bbb4c75-2tl2g    1/1     Running   0          4m51s
pod/postgresql-5945ddc6f-t2m9p   1/1     Running   0          6m20s

NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/backend      ClusterIP   10.233.56.99    <none>        5432/TCP   5m25s
service/frontend     ClusterIP   10.233.22.64    <none>        9000/TCP   3m42s
service/postgresql   ClusterIP   10.233.53.197   <none>        5432/TCP   6m20s

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/backend      1/1     1            1           6m20s
deployment.apps/frontend     1/1     1            1           4m51s
deployment.apps/postgresql   1/1     1            1           6m20s

NAME                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/backend-6599ccf48b     1         1         1       6m20s
replicaset.apps/frontend-668bbb4c75    1         1         1       4m51s
replicaset.apps/postgresql-5945ddc6f   1         1         1       6m20s
```

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
