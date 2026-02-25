# Lab 06 — Port Forward vs Proxy

## kubectl proxy

kubectl proxy

Acceso:
http://127.0.0.1:8001

Solo expone la API de Kubernetes, no aplicaciones web.

---

## port-forward

kubectl port-forward pod/apache 8080:80

Abrir:
http://localhost:8080

Formato:
LOCAL:REMOTE