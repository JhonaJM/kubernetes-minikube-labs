# Lab 02 — Pods básicos

## Crear pod nginx

kubectl run nginx1 --image=nginx

## Listar

kubectl get pods
kubectl get pods -o wide

## Describir

kubectl describe pod/nginx1