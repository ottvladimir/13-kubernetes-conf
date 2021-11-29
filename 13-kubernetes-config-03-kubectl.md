# Домашнее задание к занятию "13.3 работа с kubectl"
## Задание 1: проверить работоспособность каждого компонента
```bash
$ kubectl get pod
NAME                                  READY   STATUS    RESTARTS   AGE
backend-68bf457658-5bscf              1/1     Running   0          2m12s
frontend-5f4874c4cf-425g4             1/1     Running   0          2m12s
postgresql-5945ddc6f-gvrvl            1/1     Running   0          2m12s
```
Для проверки работы можно использовать 2 способа: port-forward и exec. Используя оба способа, проверьте каждый компонент:
* сделайте запросы к бекенду;
```bash
$ kubectl exec frontend-5f4874c4cf-425g4 -- curl -sI backend:9000
HTTP/1.1 404 Not Found
date: Mon, 29 Nov 2021 00:04:49 GMT
server: uvicorn
content-length: 22
content-type: application/json
```
```bash
$ kubectl port-forward backend-68bf457658-5bscf 9000:9000
```
```bash
$ curl -I localhost:8000
HTTP/1.1 404 Not Found
date: Mon, 29 Nov 2021 00:04:49 GMT
server: uvicorn
content-length: 22
content-type: application/json
```
* сделайте запросы к фронту;
```bash
$ kubectl exec backend-68bf457658-5bscf -- curl -I frontend:8000
HTTP/1.1 200 OK
Server: nginx/1.21.4
Date: Sun, 28 Nov 2021 23:54:56 GMT
Content-Type: text/html
Content-Length: 448
Last-Modified: Wed, 17 Nov 2021 04:35:28 GMT
Connection: keep-alive
ETag: "61948690-1c0"
Accept-Ranges: bytes
```
```bash
$ kubectl port-forward frontend-5f4874c4cf-425g4 8000:8000
```
```bash
$ curl -I localhost:8000
HTTP/1.1 200 OK
Server: nginx/1.21.4
Date: Sun, 28 Nov 2021 23:54:56 GMT
Content-Type: text/html
Content-Length: 448
Last-Modified: Wed, 17 Nov 2021 04:35:28 GMT
Connection: keep-alive
ETag: "61948690-1c0"
Accept-Ranges: bytes
```
* подключитесь к базе данных.
```bash
kubectl port-forward postgresql-5945ddc6f-gvrvl 5432:5432
```
```bash
$ psql -h localhost -p 5432 -d news -U postgres -W
Password: 
psql (13.5 (Ubuntu 13.5-0ubuntu0.21.04.1))
Type "help" for help.

news=# 
```

## Задание 2: ручное масштабирование
При работе с приложением иногда может потребоваться вручную добавить пару копий. Используя команду kubectl scale, попробуйте увеличить количество бекенда и фронта до 3. 
```bash
$ kubectl scale --replicas=3 -f backend.yml
deployment.apps/backend scaled
$ kubectl scale --replicas=3 -f frontend.yml
deployment.apps/frontend scaled
$ kubectl get pod
NAME                                  READY   STATUS    RESTARTS   AGE
backend-68bf457658-5bscf              1/1     Running   0          29m
backend-68bf457658-5qt64              1/1     Running   0          17s
backend-68bf457658-qt7cp              1/1     Running   0          17s
frontend-5f4874c4cf-425g4             1/1     Running   0          29m
frontend-5f4874c4cf-46zb6             1/1     Running   0          9s
frontend-5f4874c4cf-sfbt9             1/1     Running   0          9s
postgresql-5945ddc6f-gvrvl            1/1     Running   0          29m
$ kubectl get rs                                                                
NAME                   DESIRED   CURRENT   READY   AGE                                                                                                 
backend-68bf457658     3         3         3       30m                                                                                                 
frontend-5f4874c4cf    3         3         3       30m                                                                                                 
postgresql-5945ddc6f   1         1         1       30m                    
```
После уменьшите количество копий до 1. Проверьте, на каких нодах оказались копии после каждого действия (kubectl describe).
```bash
$ kubectl scale --replicas=1 -f frontend.yml                                    
deployment.apps/frontend scaled                                                                                                                        
$ kubectl scale --replicas=1 -f backend.yml                                     
deployment.apps/backend scaled                                           
```
```bash
$ kubectl describe pod/backend-68bf457658-5qt64
...
Events:
  Type    Reason     Age    From               Message
  ----    ------     ----   ----               -------
  Normal  Scheduled  5m56s  default-scheduler  Successfully assigned prod/backend-68bf457658-5qt64 to worker2
...
$ kubectl describe pod/frontend-5f4874c4cf-425g4
...
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  44m   default-scheduler  Successfully assigned prod/frontend-5f4874c4cf-425g4 to worker2
...
```
