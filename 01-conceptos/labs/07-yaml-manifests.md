# Lab 07 — Manifests YAML

## Crear pod con YAML

kubectl apply -f manifests/nginx.yaml

## Ver YAML generado

kubectl get pod/nginx -o yaml

## Eliminar

kubectl delete -f manifests/nginx.yaml

## Ventajas
- Infraestructura como código
- Reproducible
- Versionable