# Lab 05 — Services

## Exponer pod nginx

kubectl expose pod nginx1 --port=80 --name=nginx-svc --type=NodePort

## Obtener URL

minikube service nginx-svc

Esto abre automáticamente el navegador con la IP y puerto asignados.

## Notas
LoadBalancer no funciona localmente con Docker driver, usar NodePort.