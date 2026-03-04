# Lab 01 — Cluster Setup

## Iniciar cluster

minikube start --driver=docker

## Crear profile personalizado con múltiples nodos

minikube start -p clusterDesa --driver=docker --nodes=2

## Ver perfiles

minikube profile list

## Ver estado

minikube status

## Ver nodos

kubectl get nodes

## Detener

minikube stop

## Eliminar

minikube delete