# 🔄 Rolling Updates, Recreate y Rollbacks en Kubernetes

## 📁 Estructura del proyecto

```
📦 07-rolling-updates/
 ┣ 📄 deploy_nginx_rollingupdate.yaml
 ┗ 📄 recreate.yaml
```

---

## 🧠 ¿Qué es una estrategia de actualización?

Cuando modificas algo en los contenedores de un Deployment (imagen, puerto, variables de entorno, etc.), Kubernetes necesita reemplazar los Pods existentes por unos nuevos. La **estrategia de actualización** define **cómo** hace ese reemplazo.

> ⚠️ Las estrategias **solo se activan** cuando hay un cambio en las propiedades del contenedor (imagen, puertos, variables de entorno, etc.). Cambios en réplicas o labels no las activan.

Kubernetes ofrece dos estrategias:

| Estrategia | Descripción | ¿Downtime? |
|---|---|---|
| `RollingUpdate` | Reemplaza los Pods de forma gradual, uno a uno | ❌ No |
| `Recreate` | Borra todos los Pods y luego crea los nuevos | ✅ Sí |

---

## 1️⃣ RollingUpdate

Es la estrategia **por defecto** y la más usada en producción. Garantiza que siempre haya Pods disponibles durante la actualización.

```
ANTES:  [v1] [v1] [v1] [v1] [v1] [v1] [v1] [v1] [v1] [v1]
                    ↓ actualizando...
        [v2] [v1] [v1] [v1] [v1] [v1] [v1] [v1] [v1] [v1]
        [v2] [v2] [v1] [v1] [v1] [v1] [v1] [v1] [v1] [v1]
        ...
DESPUÉS:[v2] [v2] [v2] [v2] [v2] [v2] [v2] [v2] [v2] [v2]
```

Durante el proceso se puede observar cómo conviven Pods de la versión antigua y nueva al mismo tiempo:

```
nginx-d-5c58774b6-7mbkv   0/1  ContainerCreating   ← nueva versión creándose
nginx-d-5c58774b6-bw5h6   0/1  ContainerCreating   ← nueva versión creándose
nginx-d-688c9c9fdd-5cqsc   1/1  Running             ← versión anterior aún activa
nginx-d-688c9c9fdd-8d4zd   1/1  Running             ← versión anterior aún activa
```

### `deploy_nginx_rollingupdate.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-d
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 10
  strategy:
     type: RollingUpdate
     RollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  minReadySeconds: 10
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.17.8
        ports:
        - containerPort: 80
```

### 🔍 Parámetros del RollingUpdate

| Parámetro | Valor | Descripción |
|---|---|---|
| `maxUnavailable` | `1` | Máximo de Pods que pueden estar **caídos** durante la actualización. Con 10 réplicas, siempre habrá mínimo 9 sirviendo tráfico |
| `maxSurge` | `1` | Máximo de Pods **extra** que se pueden crear por encima del número de réplicas. Con 10 réplicas, puede haber hasta 11 Pods temporalmente |
| `minReadySeconds` | `10` | Segundos que Kubernetes espera después de que un Pod arranca antes de considerarlo estable y pasar al siguiente |

**Visualización con 10 réplicas, maxUnavailable:1, maxSurge:1:**

```
Réplicas deseadas: 10
Pods mínimos durante update: 10 - 1 = 9  (por maxUnavailable)
Pods máximos durante update: 10 + 1 = 11 (por maxSurge)

Paso 1: Crea 1 Pod nuevo (total: 11) → espera 10s (minReadySeconds)
Paso 2: Elimina 1 Pod viejo  (total: 10)
Paso 3: Crea 1 Pod nuevo (total: 11) → espera 10s
Paso 4: Elimina 1 Pod viejo  (total: 10)
... y así hasta completar los 10
```

> 💡 Sin definir estos parámetros, Kubernetes usa los valores por defecto: `maxUnavailable: 25%` y `maxSurge: 25%`, lo que significa que con 10 réplicas reemplazaría ~2-3 Pods a la vez. Definirlos explícitamente te da control total sobre la velocidad y seguridad del update.

---

## 2️⃣ Recreate

Elimina **todos** los Pods de golpe y luego crea los nuevos. Produce downtime pero es más simple y útil cuando la aplicación **no soporta dos versiones corriendo al mismo tiempo** (por ejemplo, migraciones de base de datos incompatibles entre versiones).

```
ANTES:  [v1] [v1] [v1] [v1] [v1] [v1] [v1] [v1] [v1] [v1]
                    ↓ Recreate
        [ ]  [ ]  [ ]  [ ]  [ ]  [ ]  [ ]  [ ]  [ ]  [ ]   ← ⚠️ DOWNTIME
                    ↓ creando nuevos
DESPUÉS:[v2] [v2] [v2] [v2] [v2] [v2] [v2] [v2] [v2] [v2]
```

### `recreate.yaml`

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
  replicas: 10
  strategy:
     type: Recreate
  minReadySeconds: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.17.8
        ports:
        - containerPort: 80
```

---

## 3️⃣ Historial de versiones (Revisions)

Cada vez que se aplica un cambio en el Deployment, Kubernetes guarda una nueva **revisión** en el ReplicaSet. Esto permite hacer rollbacks a cualquier versión anterior.

```bash
# Ver el historial completo de revisiones
kubectl rollout history deploy nginx-d

# Ver el detalle de una revisión específica (qué imagen usaba, labels, etc.)
kubectl rollout history deploy nginx-d --revision=1
```

**Salida ejemplo:**

```
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
3         <none>
4         <none>
5         <none>
```

> 💡 El campo `CHANGE-CAUSE` aparece como `<none>` porque no se usó la flag `--record` al aplicar los cambios. Para registrar el motivo del cambio se usa:
> ```bash
> kubectl apply -f deploy_nginx_rollingupdate.yaml --record
> ```

Cada revisión corresponde a un **ReplicaSet** diferente. Puedes verlos con:

```bash
kubectl get rs
```

```
NAME                  DESIRED   CURRENT   READY
nginx-d-5c58774b6     0         0         0      ← revisión anterior (inactiva)
nginx-d-65dfbbb4d7    10        10        10     ← revisión actual (activa)
nginx-d-688c9c9fdd    0         0         0      ← revisión anterior (inactiva)
```

> Los ReplicaSets inactivos (DESIRED: 0) se conservan para poder hacer rollback. Kubernetes no los elimina automáticamente.

---

## 4️⃣ Rollback

Si una actualización falla (imagen incorrecta, error en la app, etc.), puedes volver a una versión anterior.

### Caso real: imagen inválida

Se cambió la imagen a `nginx:1.hh` (versión inexistente), lo que causó `ImagePullBackOff`:

```
nginx-d-75d6fdf745-4ldfc   0/1   ImagePullBackOff   ← no puede descargar la imagen
nginx-d-75d6fdf745-mfpfz   0/1   ImagePullBackOff
nginx-d-65dfbbb4d7-2796t   1/1   Running            ← versión anterior sigue activa
nginx-d-65dfbbb4d7-fsqz4   1/1   Running            ← gracias al RollingUpdate
```

Gracias al `RollingUpdate`, los Pods de la versión anterior **seguían corriendo** mientras los nuevos fallaban.

### Comandos de Rollback

```bash
# Ver el estado actual del rollout
kubectl rollout status deploy nginx-d

# Ver qué imagen tiene una revisión específica
kubectl rollout history deploy nginx-d --revision=5

# Volver a la revisión anterior inmediata
kubectl rollout undo deployment nginx-d

# Volver a una revisión específica
kubectl rollout undo deployment nginx-d --to-revision=4

# rollback
kubectl rollout undo deployment nginx-d --to-revision=4 # ✅ correcta
```

### ¿Qué pasa con las revisiones después de un rollback?

Después de hacer rollback a la revisión 4, el historial cambia así:

```
ANTES del rollback:        DESPUÉS del rollback:
REVISION                   REVISION
1                          1
2                          2
3                          3
4       ← destino          3
5                          5
                           6   ← la revisión 4 se convierte en la nueva 6
```

> El rollback no "borra" la revisión destino, sino que la **reutiliza como una nueva revisión** al final del historial.

---

## 🧰 Referencia rápida de comandos

```bash
# Aplicar un deployment
kubectl apply -f deploy_nginx_rollingupdate.yaml

# Ver estado del rollout en tiempo real
kubectl rollout status deploy nginx-d

# Ver historial de revisiones
kubectl rollout history deploy nginx-d
kubectl rollout history deploy nginx-d --revision=<n>

# Rollback
kubectl rollout undo deployment nginx-d               # revisión anterior
kubectl rollout undo deployment nginx-d --to-revision=<n>  # revisión específica

# Ver ReplicaSets (uno por revisión)
kubectl get rs

# Ver pods durante el update
kubectl get pods
kubectl describe deploy nginx-d
```

---

## ✅ Conceptos clave aprendidos

- Diferencia entre **RollingUpdate** (sin downtime) y **Recreate** (con downtime)
- Cuándo se activa una estrategia de update: solo al cambiar propiedades del **contenedor**
- Qué controlan `maxUnavailable`, `maxSurge` y `minReadySeconds`
- Cómo Kubernetes mantiene **un ReplicaSet por revisión** para poder hacer rollbacks
- Cómo hacer un **rollback** a cualquier revisión del historial con `--to-revision`
- Por qué el RollingUpdate es clave en producción: los Pods viejos siguen sirviendo mientras los nuevos arrancan
- Error común: `--revision` no existe en `rollout undo`, el flag correcto es `--to-revision`
