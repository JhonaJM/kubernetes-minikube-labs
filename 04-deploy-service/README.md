# 🗂️ Aplicación Multi-Tier con Deployments y Services

## 📁 Estructura del proyecto

```
📦 04-deploy-service/
 ┣ 📄 frontend.yaml
 ┣ 📄 frontend-service.yaml
 ┣ 📄 redis-master.yaml
 ┣ 📄 redis-master-service.yaml
 ┣ 📄 redis-slave.yaml
 ┗ 📄 redis-slave-service.yaml
```

---

## 🧠 ¿Qué se construyó aquí?

En esta sección se desplegó una **aplicación Guestbook** (libro de visitas), que es una aplicación de ejemplo clásica en Kubernetes compuesta por **3 capas (tiers)**:

```
┌──────────────────────────────────────────────────────┐
│                     CLIENTE                          │
│                  (Navegador web)                     │
└───────────────────────┬──────────────────────────────┘
                        │ NodePort (puerto 80)
┌───────────────────────▼──────────────────────────────┐
│              FRONTEND (3 réplicas)                   │
│         PHP + Redis client (gb-frontend:v5)          │
│              Service: frontend (NodePort)            │
└──────────┬─────────────────────────┬─────────────────┘
           │ Escribe (puerto 6379)   │ Lee (puerto 6379)
┌──────────▼──────────┐   ┌──────────▼──────────────── ┐
│   REDIS MASTER      │   │      REDIS SLAVE (x2)      │
│   (1 réplica)       │──▶│      (2 réplicas)          │
│  Service: ClusterIP │   │   Service: ClusterIP       │
└─────────────────────┘   └────────────────────────────┘
```

> 💡 El frontend **escribe** en el Redis Master y **lee** desde los Redis Slaves. La replicación de datos de master a slaves es manejada internamente por Redis.

---

## 📦 Componentes desplegados

| Componente | Tipo | Réplicas | Imagen |
|---|---|---|---|
| `frontend` | Deployment | 3 | `gb-frontend:v5` |
| `redis-master` | Deployment | 1 | `redis` |
| `redis-slave` | Deployment | 2 | `gb-redis-follower:v2` |
| `frontend` | Service (NodePort) | — | — |
| `redis-master-svc` | Service (ClusterIP) | — | — |
| `redis-slave` | Service (ClusterIP) | — | — |

---

## 1️⃣ Redis Master

### `redis-master.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-master
  labels:
    app: redis
spec:
  selector:
    matchLabels:
      app: redis
      role: master
      tier: backend
  replicas: 1
  template:
    metadata:
      labels:
        app: redis
        role: master
        tier: backend
    spec:
      containers:
      - name: master
        image: redis
        ports:
        - containerPort: 6379
```

**Puntos clave:**
- Solo existe **1 réplica** porque Redis necesita un único nodo maestro para garantizar consistencia al escribir datos.
- Usa la imagen oficial de Redis sin modificaciones.
- Etiquetas `role: master` y `tier: backend` para diferenciarlo de los slaves.

### `redis-master-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis-master-svc
  labels:
    app: redis
    role: master
    tier: backend
spec:
  ports:
  - port: 6379
    targetPort: 6379
  selector:
    app: redis
    role: master
    tier: backend
```

**Puntos clave:**
- Tipo **ClusterIP** (por defecto cuando no se especifica `type`): solo accesible dentro del clúster.
- El `selector` apunta exactamente a los Pods del Redis Master usando las 3 labels: `app`, `role` y `tier`.
- `port` es el puerto del servicio y `targetPort` es el puerto del contenedor. Ambos son `6379`.

---

## 2️⃣ Redis Slave

### `redis-slave.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-slave
  labels:
    app: redis
spec:
  selector:
    matchLabels:
      app: redis
      role: slave
      tier: backend
  replicas: 2
  template:
    metadata:
      labels:
        app: redis
        role: slave
        tier: backend
    spec:
      containers:
      - name: slave
        image: us-docker.pkg.dev/google-samples/containers/gke/gb-redis-follower:v2
        env:
        - name: GET_HOSTS_FROM
          value: dns
```

**Puntos clave:**
- Se usan **2 réplicas** para distribuir la carga de lectura.
- La variable de entorno `GET_HOSTS_FROM: dns` le indica al slave que use el **DNS interno de Kubernetes** para localizar al master (en lugar de una IP hardcodeada). Esto es importante porque las IPs de los Pods son dinámicas.
- Usa una imagen personalizada de Google que ya incluye la lógica de replicación.

### `redis-slave-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis-slave
  labels:
    app: redis
    role: slave
    tier: backend
spec:
  ports:
  - port: 6379
  selector:
    app: redis
    role: slave
    tier: backend
```

**Puntos clave:**
- También es tipo **ClusterIP**, accesible solo dentro del clúster.
- El frontend usará este servicio para las operaciones de **lectura**.

---

## 3️⃣ Frontend

### `frontend.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  labels:
    app: guestbook
spec:
  selector:
    matchLabels:
      app: guestbook
      tier: frontend
  replicas: 3
  template:
    metadata:
      labels:
        app: guestbook
        tier: frontend
    spec:
      containers:
      - name: php-redis
        image: us-docker.pkg.dev/google-samples/containers/gke/gb-frontend:v5
        env:
        - name: GET_HOSTS_FROM
          value: dns
```

**Puntos clave:**
- Se usan **3 réplicas** para alta disponibilidad: si un Pod cae, los otros dos siguen sirviendo tráfico.
- Al igual que el slave, usa `GET_HOSTS_FROM: dns` para localizar los servicios de Redis por nombre en lugar de por IP.
- El nombre de contenedor `php-redis` indica que esta imagen tiene PHP y la librería cliente de Redis integrados.

### `frontend-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend
  labels:
    app: guestbook
    tier: frontend
spec:
  type: NodePort
  #type: LoadBalancer
  ports:
  - port: 80
  selector:
    app: guestbook
    tier: frontend
```

**Puntos clave:**
- Es el **único servicio expuesto al exterior** de toda la aplicación.
- Tiene comentado `type: LoadBalancer`, pensado para ser usado en cloud (AWS, GCP, Azure). En local con Minikube se usa `NodePort`.
- El `selector` usa las labels `app: guestbook` y `tier: frontend` para enrutar tráfico solo a los Pods del frontend.

---

## ⚠️ Detalle crítico: Los nombres de los Services NO son arbitrarios

Este es un punto que fácilmente se pasa por alto y es clave para entender por qué todo funciona.

La imagen `gb-frontend:v5` contiene código PHP que tiene **hardcodeados** los nombres de los Services a los que se conecta:

```php
// ESCRIBE → busca un Service llamado EXACTAMENTE "redis-master"
$host = 'redis-master';
$client->connect($host, 6379);
$client->set($key, $value);

// LEE → busca un Service llamado EXACTAMENTE "redis-slave"
$host = 'redis-slave';
$client->connect($host, 6379);
return $client->get($key);
```

Cuando el frontend intenta conectarse a `redis-master`, Kubernetes busca en su DNS interno un Service con **ese nombre exacto**. Si en tu YAML hubieras puesto `name: redis-master-test`, el DNS no encontraría nada y la conexión fallaría:

```
PHP busca "redis-master"
        ↓
DNS de Kubernetes busca un Service llamado "redis-master"
        ↓
No existe → ❌ CONNECTION REFUSED
```

### La regla de oro

```
metadata:              ==   $host = 'redis-master';
  name: redis-master
```

El nombre del Service en el YAML debe coincidir **exactamente** con el nombre que el código usa para conectarse. En este caso Google construyó la imagen con esos nombres fijos, y nuestros YAMLs deben respetarlos.

> 💡 Cuando seas tú quien escribe tanto el código como los YAMLs, puedes usar los nombres que quieras — siempre que sean iguales en ambos lados. El DNS interno de Kubernetes resuelve el nombre del Service a su IP automáticamente, igual que un hostname en cualquier aplicación web.

---

## ▶️ Despliegue de la aplicación

Se recomienda desplegar en este orden para respetar las dependencias:

```bash
# 1. Desplegar el Redis Master
kubectl apply -f redis-master.yaml
kubectl apply -f redis-master-service.yaml

# 2. Desplegar los Redis Slaves
kubectl apply -f redis-slave.yaml
kubectl apply -f redis-slave-service.yaml

# 3. Desplegar el Frontend
kubectl apply -f frontend.yaml
kubectl apply -f frontend-service.yaml
```

O de una sola vez apuntando a toda la carpeta:

```bash
kubectl apply -f .
```

---

## ✅ Verificar el despliegue

```bash
# Ver todos los recursos creados
kubectl get all

# Ver solo deployments
kubectl get deploy

# Ver pods con sus labels
kubectl get pods --show-labels

# Ver servicios
kubectl get svc

# Abrir la aplicación en el navegador (Minikube)
minikube service frontend
```

**Resultado esperado:**

```
NAME                            READY   STATUS    RESTARTS   AGE
pod/frontend-xxx                1/1     Running   0          1m
pod/frontend-yyy                1/1     Running   0          1m
pod/frontend-zzz                1/1     Running   0          1m
pod/redis-master-xxx            1/1     Running   0          2m
pod/redis-slave-xxx             1/1     Running   0          1m
pod/redis-slave-yyy             1/1     Running   0          1m

NAME                    TYPE        CLUSTER-IP     PORT(S)        AGE
service/frontend        NodePort    10.x.x.x       80:3xxxx/TCP   1m
service/redis-master-svc ClusterIP  10.x.x.x       6379/TCP       2m
service/redis-slave     ClusterIP   10.x.x.x       6379/TCP       1m
```

---

## 🧹 Eliminar todos los recursos

```bash
kubectl delete -f .
```

---

## 🔑 Conceptos clave aprendidos

- Cómo diseñar una **arquitectura multi-tier** en Kubernetes
- Por qué el **Redis Master tiene 1 réplica** y los **Slaves tienen 2**
- Cómo los servicios usan **múltiples labels en el selector** para mayor precisión
- La diferencia entre `port` y `targetPort` en un Service
- Uso de la variable `GET_HOSTS_FROM: dns` para descubrimiento de servicios por nombre
- Cuándo usar **NodePort** (local/Minikube) vs **LoadBalancer** (cloud)
- Cómo exponer **solo la capa necesaria** al exterior (solo el frontend) manteniendo el backend privado con ClusterIP
