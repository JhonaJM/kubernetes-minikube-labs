# Lab 03 — Debugging dentro del contenedor

## Ejecutar comandos

kubectl exec nginx1 -- ls
kubectl exec nginx1 -- uname -a

## Entrar al contenedor

kubectl exec -it nginx1 -- bash

## Ver logs

kubectl logs apache

## Instalar herramientas

apt-get update
apt-get install wget