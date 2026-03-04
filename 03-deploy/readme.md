1.- deployment
necesito que eso sea otro repo, empezamos con deployments asi que vamos a documentar todo

--modo imperativo

C:\Users\jho_j>kubectl create deployment apachedeployment --image=httpd
deployment.apps/apachedeployment created
C:\Users\jho_j>kubectl get deploy
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
apachedeployment   1/1     1            1           42s
C:\Users\jho_j>kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
apachedeployment-79dc7bfdbb-p8gbt   1/1     Running   0          3m44s
C:\Users\jho_j>kubectl describe deploy apachedeployment

--modo declarativa (desde un archi yaml)
C:\Users\jho_j\Documents\studies\kubernetes\deployments>kubectl apply -f deploy_nginx.yaml
deployment.apps/nginx-d created

C:\Users\jho_j\Documents\studies\kubernetes\deployments>kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
apachedeployment-79dc7bfdbb-p8gbt   1/1     Running   0          23m
nginx-d-59f86b59ff-2x6k2            1/1     Running   0          29s
nginx-d-59f86b59ff-98tsc            1/1     Running   0          30s

C:\Users\jho_j\Documents\studies\kubernetes\deployments>kubectl get deploy
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
apachedeployment   1/1     1            1           24m
nginx-d            2/2     2            2           59s

C:\Users\jho_j\Documents\studies\kubernetes\deployments>kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
apachedeployment-79dc7bfdbb   1         1         1       24m
nginx-d-59f86b59ff            2         2         2       83s

C:\Users\jho_j\Documents\studies\kubernetes\deployments>kubectl get pods -l app=nginx
NAME                       READY   STATUS    RESTARTS   AGE
nginx-d-59f86b59ff-2x6k2   1/1     Running   0          3m4s
nginx-d-59f86b59ff-98tsc   1/1     Running   0          3m5s

C:\Users\jho_j\Documents\studies\kubernetes\deployments>kubectl edit deploy nginx-d
C:\Users\jho_j\Documents\studies\kubernetes\deployments>kubectl scale deploy nginx-d --replicas=5
deployment.apps/nginx-d scaled

C:\Users\jho_j\Documents\studies\kubernetes\deployments>kubectl scale deploy nginx-d --replicas=5
deployment.apps/nginx-d scaled

C:\Users\jho_j\Documents\studies\kubernetes\deployments>kubectl get deploy
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
apachedeployment   1/1     1            1           38m
nginx-d            5/5     5            5           15m

2.- servicios (como conectarnos a los recursos)
temas a tratar:
que es un servicio
tipos de srecicios
crear un servicio
gestionar y configurar un servicio
rolling updates
rollbacks
recreate

introduccion: el deployment genera tiene dentro de si mismo los pods, pero el cliente no puede acceder a ellos desde fuera porque no sabemos su ip ni nada, entonces el servicio es el puente entre estos 2

tipos de servicios:
cluster ip : accesible solo desde dentro del cluster
nodeport: accesible desde fuera del cluster
loadbalancer: accesible desde fuera del cluster, integrado por tercedos: aws, azure, gcp

como detecta los servicios a los pods y como sabe si se caen o no? el servicio usa los selectors para poder identificar 

crear servicio tipo node port
C:\Users\jho_j>kubectl config current-context
minikube
C:\Users\jho_j>minikube start

C:\Users\jho_j>kubectl create deployment apache1 --image=httpd
deployment.apps/apache1 created

C:\Users\jho_j>kubectl get all
NAME                                    READY   STATUS    RESTARTS        AGE
pod/apache1-84b77df898-kwj72            1/1     Running   0               65s
pod/apachedeployment-79dc7bfdbb-p8gbt   1/1     Running   1 (3m18s ago)   37h
pod/nginx-d-59f86b59ff-2x6k2            1/1     Running   1 (3m18s ago)   37h
pod/nginx-d-59f86b59ff-98tsc            1/1     Running   1 (3m18s ago)   37h

NAME                 TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/kubernetes   ClusterIP      10.96.0.1       <none>        443/TCP        9d
service/nginx-svc    LoadBalancer   10.102.47.151   <pending>     80:30379/TCP   4d21h

NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/apache1            1/1     1            1           65s
deployment.apps/apachedeployment   1/1     1            1           37h
deployment.apps/nginx-d            2/2     2            2           37h

NAME                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/apache1-84b77df898            1         1         1       65s
replicaset.apps/apachedeployment-79dc7bfdbb   1         1         1       37h
replicaset.apps/nginx-d-59f86b59ff            2         2         2       37h

C:\Users\jho_j>kubectl expose deploy apache1 --port=80 --type=NodePort
service/apache1 exposed

C:\Users\jho_j>minikube service apache1
┌───────────┬─────────┬─────────────┬───────────────────────────┐
│ NAMESPACE │  NAME   │ TARGET PORT │            URL            │
├───────────┼─────────┼─────────────┼───────────────────────────┤
│ default   │ apache1 │ 80          │ http://192.168.49.2:30890 │
└───────────┴─────────┴─────────────┴───────────────────────────┘
* Starting tunnel for service apache1.
┌───────────┬─────────┬─────────────┬────────────────────────┐
│ NAMESPACE │  NAME   │ TARGET PORT │          URL           │
├───────────┼─────────┼─────────────┼────────────────────────┤
│ default   │ apache1 │             │ http://127.0.0.1:54994 │
└───────────┴─────────┴─────────────┴────────────────────────┘
* Opening service default/apache1 in default browser...
! Porque estás usando controlador Docker en windows, la terminal debe abrirse para ejecutarlo

--crear servicio tipo clusterIP

C:\Users\jho_j>kubectl create deployment redis-master --image=redis
deployment.apps/redis-master created

C:\Users\jho_j>kubectl create deployment redis-cli --image=redis
deployment.apps/redis-cli created

C:\Users\jho_j>kubectl expose deploy redis-master --port=6379 --type=ClusterIP
service/redis-master exposed


C:\Users\jho_j>kubectl exec redis-cli-8567864475-9zs64 -it -- bash
root@redis-cli-8567864475-9zs64:/data#

C:\Users\jho_j>kubectl exec redis-cli-8567864475-9zs64 -it -- bash
root@redis-cli-8567864475-9zs64:/data# redis-cli -h redis-master
redis-master:6379> set v1 10
OK
redis-master:6379> get v1
"10"
redis-master:6379>

PS C:\Users\jho_j> kubectl get pod
NAME                            READY   STATUS    RESTARTS   AGE
redis-cli-8567864475-9zs64      1/1     Running   0          5m28s
redis-master-7fbcfd8f8c-525b9   1/1     Running   0          8m7s
PS C:\Users\jho_j> kubectl exec -it redis-master-7fbcfd8f8c-525b9 -- bash
root@redis-master-7fbcfd8f8c-525b9:/data# redis-cli
127.0.0.1:6379> info keyspace
# Keyspace
db0:keys=1,expires=0,avg_ttl=0,subexpiry=0
127.0.0.1:6379> get v1
"10"
127.0.0.1:6379>
