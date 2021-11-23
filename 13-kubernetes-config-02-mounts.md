# Домашнее задание к занятию "13.2 разделы и монтирование"

## Задание 1: подключить для тестового конфига общую папку
В stage окружении часто возникает необходимость отдавать статику бекенда сразу фронтом. Проще всего сделать это через общую папку. Требования:
* в поде подключена общая папка между контейнерами (например, /static);
```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: myapp
  name: frontend-backend-volume
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
        volumeMounts:
          - mountPath: /static
            name: page
      - image: ottvladimir/backend:main
        name: backend
        ports:
        - containerPort: 9000
        volumeMounts:
          - mountPath: /static
            name: page
      volumes:
        - name: page
        emptyDir: {}
   ```
* после записи чего-либо в контейнере с беком файлы можно получить из контейнера с фронтом.
```bash
$ kubectl exec -it frontend-backend-volume-7ff5f9f675-975n5 -c backend -- bash
root@frontend-backend-volume-7ff5f9f675-975n5:/app# cd /static/
root@frontend-backend-volume-7ff5f9f675-975n5:/static# echo Page >> testpage.html
^PQ
$ kubectl exec -it frontend-backend-volume-7ff5f9f675-975n5 -c frontend -- bash
root@frontend-backend-volume-7ff5f9f675-975n5:/app# cat /static/testpage.html 
Page
```

## Задание 2: подключить общую папку для прода
Поработав на stage, доработки нужно отправить на прод. В продуктиве у нас контейнеры крутятся в разных подах, поэтому потребуется PV и связь через PVC. Сам PV должен быть связан с NFS сервером.   
pv.yml
```yml
---
kind: PersistentVolume
apiVersion: v1
metadata:
  name: pv-static
spec:
  storageClassName: nfs
  accessModes:
    - ReadWriteMany
  capacity:
       storage: 1Gi
  hostPath:
    path: /tmp/pv
```
pvc.yml
```yml
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pvc
spec:
  storageClassName: nfs
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
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
        volumeMounts:
          - mountPath: /static
            name: pv
      volumes:
        - name: pv
          persistentVolumeClaim:
            claimName: pvc
```
frontend.yml
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
          volumeMounts:
            - mountPath: /static
              name: pv
      volumes: 
        - name: pv
          persistentVolumeClaim:
            claimName: pvc
```
```bash
$ kubectl get all
NAME                                           READY   STATUS    RESTARTS   AGE
pod/backend-76fdc5c599-csqbq                   1/1     Running   0          6m28s
pod/frontend-6d956bfc8-zk7c5                   1/1     Running   0          6m28s
pod/frontend-backend-volume-7ff5f9f675-975n5   2/2     Running   0          129m
pod/postgresql-5945ddc6f-hh4hj                 1/1     Running   0          6m28s

NAME                 TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
service/backend      ClusterIP   10.233.2.17    <none>        9000/TCP   6m28s
service/frontend     ClusterIP   10.233.23.47   <none>        8080/TCP   6m28s
service/postgresql   ClusterIP   10.233.63.13   <none>        5432/TCP   6m28s

NAME                                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/backend                   1/1     1            1           6m28s
deployment.apps/frontend                  1/1     1            1           6m28s
deployment.apps/frontend-backend-volume   1/1     1            1           129m
deployment.apps/postgresql                1/1     1            1           6m28s

NAME                                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/backend-76fdc5c599                   1         1         1       6m28s
replicaset.apps/frontend-6d956bfc8                   1         1         1       6m28s
replicaset.apps/frontend-backend-volume-7ff5f9f675   1         1         1       129m
replicaset.apps/postgresql-5945ddc6f                 1         1         1       6m28s

$ kubectl get pv
NAME        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM        STORAGECLASS   REASON   AGE
pv-static   1Gi        RWX            Retain           Bound    volume/pvc   nfs                     6m58s
$ kubectl get pvc
NAME   STATUS   VOLUME      CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc    Bound    pv-static   1Gi        RWX            nfs            4m9s           
```

* файлы, созданные бекендом, должны быть доступны фронту.

```bash
$ kubectl exec -it pod/backend-76fdc5c599-csqbq -- bash                                                                                                                    
root@backend-76fdc5c599-csqbq:/app# touch /static/Testfile                                                               
root@backend-76fdc5c599-csqbq:/app# exit                                                                                 
```
```bash
$ kubectl exec -it pod/frontend-6d956bfc8-zk7c5 -- bash                                                                                                           root@frontend-6d956bfc8-zk7c5:/app# ls /static/                                                                          
Testfile                                                              
```
