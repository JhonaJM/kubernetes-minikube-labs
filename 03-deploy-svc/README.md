# 🚀 Deployments y Servicios en Kubernetes

## 📁 Estructura del proyecto

```
📦 03-deploy-svc/
    ┗ 📄 deploy_nginx.yaml
```

---

## ☸️ ¿Qué es un Deployment?

Un **Deployment** es un objeto de Kubernetes que gestiona el ciclo de vida de los Pods de forma declarativa. A diferencia de crear Pods a mano, el Deployment se encarga de:

- Mantener el número deseado de réplicas en ejecución
- Reemplazar automáticamente Pods caídos
- Gestionar actualizaciones y rollbacks de forma controlada
- Crear y gestionar internamente un **ReplicaSet**

> 📌 La jerarquía es: `Deployment → ReplicaSet → Pods`

---

## 1️⃣ Crear un Deployment — Modo Imperativo

El modo imperativo permite crear recursos directamente desde la terminal sin necesidad de un archivo YAML.

```bash
kubectl create deployment apachedeployment --image=httpd
```

Verificar que se creó correctamente:

```bash
# Ver los deployments activos
kubectl get deploy

# Ver los pods generados por el deployment
kubectl get pods

# Ver información detallada del deployment
kubectl describe deploy apachedeployment
```

**Salida esperada:**

```
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
apachedeployment   1/1     1            1           42s
```

---

## 2️⃣ Crear un Deployment — Modo Declarativo (YAML)

El modo declarativo define el estado deseado en un archivo YAML y lo aplica al clúster. Es el método recomendado en entornos reales.

### 📄 `deploy_nginx.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-d
  labels:
   estado: "1"
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

### 🔍 Explicación de los campos

| Campo | Descripción |
|---|---|
| `apiVersion: apps/v1` | Versión de la API para Deployments (desde Kubernetes 1.9+) |
| `kind: Deployment` | Tipo de recurso a crear |
| `metadata.name` | Nombre del Deployment |
| `metadata.labels` | Etiquetas del Deployment (útil para organización) |
| `spec.selector.matchLabels` | Selector que vincula el Deployment con sus Pods |
| `spec.replicas` | Número de Pods que se desean mantener en ejecución |
| `spec.template` | Plantilla que define cómo será cada Pod creado |
| `template.metadata.labels` | Labels de los Pods — deben coincidir con `matchLabels` |
| `containers.image` | Imagen Docker a utilizar |
| `containers.ports.containerPort` | Puerto que expone el contenedor |

### ▶️ Aplicar el manifiesto

```bash
kubectl apply -f deploy_nginx.yaml
```

**Verificar el resultado:**

```bash
kubectl get pods
kubectl get deploy
kubectl get rs   # Ver los ReplicaSets creados automáticamente
```

```
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
apachedeployment   1/1     1            1           24m
nginx-d            2/2     2            2           59s
```

**Filtrar pods por label:**

```bash
kubectl get pods -l app=nginx
```

---

## 3️⃣ Escalar un Deployment

Se puede modificar el número de réplicas de dos formas:

**De forma imperativa (desde terminal):**

```bash
kubectl scale deploy nginx-d --replicas=5
```

**De forma declarativa (editando en vivo):**

```bash
kubectl edit deploy nginx-d
# Se abre el editor, modificar spec.replicas y guardar
```

```
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-d            5/5     5            5           15m
```

---

## 🔌 ¿Qué es un Servicio?

Los Pods generados por un Deployment **no son accesibles directamente** desde el exterior del clúster. No conocemos su IP, y además son efímeros: pueden morir y renacer con una IP diferente.

Un **Service** actúa como puente estable entre los clientes y los Pods, independientemente de cuántos haya o dónde estén corriendo.

> El servicio usa **selectors (labels)** para identificar a qué Pods debe enrutar el tráfico, y detecta automáticamente si un Pod cae o se crea uno nuevo.

---

## 4️⃣ Tipos de Servicios

| Tipo | Accesible desde | Descripción |
|---|---|---|
| `ClusterIP` | Solo dentro del clúster | Tipo por defecto. Comunicación interna entre servicios |
| `NodePort` | Desde fuera del clúster | Expone el servicio en un puerto del nodo |
| `LoadBalancer` | Desde fuera del clúster | Integrado con proveedores cloud (AWS, Azure, GCP) |

---

## 5️⃣ Servicio tipo NodePort — Ejemplo con Apache

Permite acceder al deployment **desde fuera del clúster**.

```bash
# Crear el deployment
kubectl create deployment apache1 --image=httpd

# Exponer el deployment con NodePort en el puerto 80
kubectl expose deploy apache1 --port=80 --type=NodePort

# En Minikube, abrir el servicio en el navegador
minikube service apache1
```

**Salida de Minikube:**

```
┌───────────┬─────────┬─────────────┬───────────────────────────┐
│ NAMESPACE │  NAME   │ TARGET PORT │            URL            │
├───────────┼─────────┼─────────────┼───────────────────────────┤
│ default   │ apache1 │ 80          │ http://192.168.49.2:30890 │
└───────────┴─────────┴─────────────┴───────────────────────────┘
```

> ⚠️ En Windows con driver Docker, Minikube crea un túnel automático y abre el navegador en `http://127.0.0.1:<puerto>`.

---

## 6️⃣ Servicio tipo ClusterIP — Ejemplo con Redis

Permite la **comunicación interna** entre Pods dentro del clúster. En este ejemplo, un pod cliente de Redis se comunica con un pod servidor de Redis usando su nombre de servicio como hostname.

```bash
# Crear el servidor Redis
kubectl create deployment redis-master --image=redis

# Crear el cliente Redis
kubectl create deployment redis-cli --image=redis

# Exponer el servidor con ClusterIP en el puerto 6379
kubectl expose deploy redis-master --port=6379 --type=ClusterIP
```

**Conectarse desde el cliente al servidor:**

```bash
# Acceder al pod cliente
kubectl exec redis-cli-8567864475-9zs64 -it -- bash

# Conectarse al servidor usando su nombre de servicio
redis-cli -h redis-master

# Guardar y leer un valor
redis-master:6379> set v1 10
OK
redis-master:6379> get v1
"10"
```

**Verificar desde el pod servidor:**

```bash
kubectl exec -it redis-master-7fbcfd8f8c-525b9 -- bash

redis-cli
127.0.0.1:6379> get v1
"10"
```

> 💡 Kubernetes proporciona **DNS interno** al clúster, por lo que los servicios son accesibles por su nombre (`redis-master`) sin necesidad de conocer su IP.

---

## 🧰 Comandos de referencia rápida

```bash
# Crear deployment
kubectl create deployment <nombre> --image=<imagen>
kubectl apply -f <archivo>.yaml

# Consultar recursos
kubectl get deploy
kubectl get pods
kubectl get rs
kubectl get all
kubectl get pods -l <label>=<valor>

# Escalar
kubectl scale deploy <nombre> --replicas=<n>

# Editar en vivo
kubectl edit deploy <nombre>

# Exponer como servicio
kubectl expose deploy <nombre> --port=<puerto> --type=<tipo>

# Acceder a un pod (shell interactivo)
kubectl exec -it <pod> -- bash

# Ver detalles de un recurso
kubectl describe deploy <nombre>

# Eliminar recursos
kubectl delete deploy <nombre>
kubectl delete -f <archivo>.yaml
```

---

## ✅ Conceptos clave aprendidos

- Qué es un Deployment y cómo gestiona Pods y ReplicaSets
- Diferencia entre el **modo imperativo** y el **modo declarativo**
- Cómo escalar réplicas de forma dinámica
- Qué es un Servicio y por qué es necesario
- Tipos de servicios: **ClusterIP**, **NodePort** y **LoadBalancer**
- Cómo los servicios usan **labels/selectors** para descubrir Pods
- Comunicación interna entre Pods mediante **DNS de Kubernetes**
