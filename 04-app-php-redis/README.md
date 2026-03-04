# 🌐 Despliegue de aplicación web propia con Apache y Kubernetes

## 📁 Estructura del proyecto

```
📦 04-app-php-redis/
 ┣ 📂 web/
 ┃  ┣ 📄 index.html
 ┃  ┣ 📂 css/
 ┃  ┣ 📂 fonts/
 ┃  ┣ 📂 img/
 ┃  ┗ 📂 js/
 ┣ 📄 Dockerfile
 ┣ 📄 deploy_web.yaml
 ┣ 📄 web-svc.yaml
 ┗ 📄 completo.yaml
```

---

## 🧠 ¿Qué se hizo en este capítulo?

Este capítulo marca un paso importante: en lugar de usar imágenes de terceros (como `nginx` o `httpd`), aquí se construyó una **imagen Docker propia** a partir de una aplicación web real (HTML + CSS + JS), se publicó en Docker Hub, y luego se desplegó en Kubernetes usando un Deployment y un Service.

El flujo completo fue:

```
Código web (HTML/CSS/JS)
        ↓
   Dockerfile → imagen Docker propia
        ↓
   Docker Hub (jhonajm/web-practice:latest)
        ↓
   Kubernetes Deployment → Pods con Apache
        ↓
   Kubernetes Service (NodePort) → acceso desde el navegador
```

---

## 🐳 Dockerfile — Construyendo la imagen propia

```dockerfile
FROM ubuntu:22.04
LABEL maintainer="jho.jim.01@gmail.com"

RUN apt-get update
RUN apt-get install -y apache2

ADD web /var/www/html

EXPOSE 80

CMD ["apachectl", "-D", "FOREGROUND"]
```

### 🔍 Explicación paso a paso

| Instrucción | Descripción |
|---|---|
| `FROM ubuntu:22.04` | Usa una versión **específica** de Ubuntu (buena práctica: evita sorpresas con actualizaciones) |
| `LABEL maintainer` | Metadato que identifica al autor de la imagen |
| `RUN apt-get update` | Actualiza los repositorios del sistema |
| `RUN apt-get install -y apache2` | Instala el servidor web Apache |
| `ADD web /var/www/html` | Copia toda la carpeta `web/` local al directorio raíz de Apache dentro del contenedor |
| `EXPOSE 80` | Documenta que el contenedor escucha en el puerto 80 |
| `CMD [...]` | Arranca Apache en primer plano (necesario para que el contenedor no se cierre) |

> 💡 **`ADD` vs `COPY`**: Ambos copian archivos al contenedor, pero `ADD` tiene superpoderes extra: puede descomprimir `.tar` automáticamente y aceptar URLs. Para copiar archivos locales simples, `COPY` es la opción más explícita y recomendada. Aquí se usó `ADD` porque el curso lo empleaba así.

> 💡 **`CMD` vs `ENTRYPOINT`**: A diferencia del capítulo anterior donde se usó `ENTRYPOINT` (que no puede sobreescribirse), aquí se usa `CMD`, que sí puede ser sobreescrito al crear el contenedor. Para arrancar servicios en producción, `ENTRYPOINT` es más seguro.

---

## 🔨 Build, prueba local y push de la imagen

```bash
# 1. Construir la imagen
docker build -t jhonajm/web-practice .

# 2. Probar localmente antes de subir a Kubernetes
docker container run --name web1 -d -p 9090:80 jhonajm/web-practice:latest
# Abrir http://localhost:9090 para verificar que la web funciona

# 3. Publicar en Docker Hub
docker push jhonajm/web-practice:latest
```

> ✅ Siempre es buena práctica **probar el contenedor localmente** con `docker run` antes de desplegarlo en Kubernetes. Así separas los posibles errores: si falla en Docker es problema de la imagen, si falla en Kubernetes es problema del manifiesto.

---

## ☸️ Archivos de Kubernetes

Este capítulo introduce dos formas de organizar los manifiestos YAML: **archivos separados** y **un único archivo combinado**.

### Opción A — Archivos separados

#### `deploy_web.yaml` — Solo el Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-d
spec:
  selector:
    matchLabels:
      app: web
  replicas: 2
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: apache
        image: jhonajm/web-practice:latest
        ports:
        - containerPort: 80
```

#### `web-svc.yaml` — Solo el Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-svc
  labels:
     app: web
spec:
  type: NodePort
  ports:
  - port: 80
    nodePort: 30002
    protocol: TCP
  selector:
     app: web
```

```bash
# Aplicar por separado
kubectl apply -f deploy_web.yaml
kubectl apply -f web-svc.yaml
```

---

### Opción B — Archivo único `completo.yaml`

Kubernetes permite combinar múltiples recursos en un solo archivo YAML separándolos con `---`:

```yaml
# DEPLOYMENT
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-d
# ... resto del deployment ...
---
# SERVICE
apiVersion: v1
kind: Service
metadata:
  name: web-svc
# ... resto del service ...
```

```bash
# Aplicar todo de una sola vez
kubectl apply -f completo.yaml
```

> 💡 El separador `---` le indica a Kubernetes que lo que viene a continuación es un recurso independiente dentro del mismo archivo. Es útil para mantener recursos relacionados juntos y desplegarlos con un solo comando.

---

## 🔍 Explicación de campos clave del Service

| Campo | Valor | Descripción |
|---|---|---|
| `type` | `NodePort` | Expone el servicio fuera del clúster |
| `port` | `80` | Puerto del Service dentro del clúster |
| `nodePort` | `30002` | Puerto fijo en el nodo para acceso externo |
| `protocol` | `TCP` | Protocolo de red |
| `selector.app` | `web` | Enruta tráfico a los Pods con label `app: web` |

> ⚠️ Los `nodePort` válidos están en el rango **30000–32767**. Si no se especifica, Kubernetes asigna uno aleatorio. Aquí se fijó el `30002` para tener siempre la misma URL de acceso.

---

## ▶️ Comandos utilizados

```bash
# Desplegar todo junto
kubectl apply -f completo.yaml

# Ver los endpoints que el Service está usando (a qué Pods apunta)
kubectl get endpoints web-svc -o yaml

# Ver todos los recursos creados
kubectl get all

# Acceder desde Minikube
minikube service web-svc
```

### ¿Qué es un Endpoint?

Cuando un Service encuentra Pods que coinciden con su `selector`, Kubernetes crea automáticamente un objeto **Endpoint** que contiene las IPs reales de esos Pods. Es la forma en que el Service sabe a dónde enviar el tráfico:

```
Service (web-svc)
    ↓ selector: app=web
Endpoints (IPs de los Pods)
    ↓
Pod 1 (10.244.x.x:80)
Pod 2 (10.244.x.x:80)
```

```bash
# Ver los endpoints en detalle
kubectl get endpoints web-svc -o yaml
```

---

## ✅ Resultado esperado

Accediendo al puerto `30002` del nodo (o via `minikube service web-svc`) deberías ver la aplicación web propia sirviendo los archivos HTML/CSS/JS de la carpeta `web/`.

```
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
web-d     2/2     2            2           1m

NAME       TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
web-svc    NodePort   10.x.x.x       <none>        80:30002/TCP   1m
```

---

## 🔑 Conceptos clave aprendidos

- Cómo construir una **imagen Docker propia** a partir de una aplicación web real
- Diferencia entre `ADD` y `COPY` en un Dockerfile
- Diferencia entre `CMD` y `ENTRYPOINT`
- Buena práctica: **probar localmente con Docker** antes de desplegar en Kubernetes
- Cómo definir múltiples recursos en un **único archivo YAML** usando `---`
- Cómo fijar un `nodePort` específico para tener una URL de acceso estable
- Qué son los **Endpoints** y cómo los usa el Service para encontrar los Pods
