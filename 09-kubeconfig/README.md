# Configuración del Cluster: Kubeconfig

Este laboratorio explica cómo Kubernetes gestiona la configuración de acceso a los clusters mediante el fichero `kubeconfig`. Se cubren todos los subcomandos de `kubectl config` para inspeccionar, crear y cambiar clusters, usuarios y contextos.

## Estructura del laboratorio

```
09-kubeconfig/
├── README.md                  ← este fichero
└── ejemplo-kubeconfig.yaml    ← estructura completa y comentada del fichero kubeconfig
```

---

## 01 — ¿Qué es Kubeconfig?

`kubeconfig` es el fichero YAML que `kubectl` usa para saber:
- **A qué cluster** conectarse (URL del API Server, certificados)
- **Con qué identidad** (usuario, token, certificados de cliente)
- **En qué contexto** operar (combinación de cluster + usuario + namespace)

### Dónde se encuentra por defecto

| Sistema Operativo | Ruta |
|-------------------|------|
| Linux / Mac | `~/.kube/config` |
| Windows | `%USERPROFILE%\.kube\config` → `C:\Users\jho_j\.kube\config` |

### Cómo indicar qué fichero usar

kubectl busca el fichero de configuración en este orden — usa el primero que encuentre:

**1. Flag `--kubeconfig` en el comando**

Afecta solo a ese comando concreto. El flag va siempre después de `kubectl` y antes del subcomando:

```bash
# Ver el config por defecto (~/.kube/config):
kubectl config view

# Ver un fichero de config específico:
kubectl --kubeconfig=C:\Users\jho_j\.kube\config_alternativo config view
```

**2. Variable de entorno `$KUBECONFIG`**

Útil cuando quieres trabajar con un fichero distinto durante toda la sesión del terminal sin escribir `--kubeconfig` en cada comando:

```bash
# Windows PowerShell — activar para toda la sesión:
$env:KUBECONFIG = "C:\Users\jho_j\.kube\config_alternativo"
kubectl config view      # ahora muestra config_alternativo, sin necesidad de --kubeconfig
kubectl get pods         # este comando también usará config_alternativo

# Cuando termines, volver al comportamiento por defecto:
Remove-Item Env:\KUBECONFIG

# Linux / Mac:
export KUBECONFIG=~/.kube/config_alternativo
kubectl config view
unset KUBECONFIG         # volver al por defecto
```

**3. Fichero por defecto**

Si no usas ni `--kubeconfig` ni `$KUBECONFIG`, kubectl usa automáticamente el fichero en su ubicación estándar:

```bash
# Simplemente ejecutar kubectl sin nada extra usa ~/.kube/config (o el equivalente en Windows)
kubectl config view
```

---

## 02 — Estructura del fichero kubeconfig

El fichero kubeconfig tiene cuatro bloques principales. Ver `ejemplo-kubeconfig.yaml` para la estructura completa.

```
apiVersion: v1
kind: Config
clusters:    [...]    ← dónde están los clusters y sus certificados
users:       [...]    ← credenciales de autenticación del cliente
contexts:    [...]    ← combinaciones cluster + usuario + namespace
current-context: xxx  ← contexto activo por defecto
```

### Bloque `clusters`

Define los **endpoints** de cada cluster de Kubernetes.

```yaml
clusters:
  - name: produccion               # nombre clave (se referencia desde los contextos)
    cluster:
      server: https://192.168.1.100:6443   # URL completa del API Server
      certificate-authority: /etc/kubernetes/pki/ca.crt  # CA del cluster
      # insecure-skip-tls-verify: true     # omitir verificación TLS (solo pruebas)
```

- `server` — URL completa del API Server de Kubernetes
- `certificate-authority` — ruta al certificado de la CA que firmó el certificado del API Server
- `insecure-skip-tls-verify: true` — útil en entornos de laboratorio donde el certificado no está firmado por una CA de confianza del sistema. **Nunca en producción.**
- Se gestiona con: `kubectl config set-cluster`

### Bloque `users`

Define las **credenciales del cliente** para autenticarse ante el API Server.

```yaml
users:
  - name: desarrollo1              # nombre clave
    user:
      token: eyJhbGci...           # token JWT (ServiceAccount, OIDC)

  - name: desarrollo2
    user:
      client-certificate: /ruta/cert.crt  # certificado mTLS del cliente
      client-key: /ruta/cert.key          # clave privada correspondiente
```

| Tipo de credencial | Campo(s) | Notas |
|--------------------|----------|-------|
| Token JWT | `token` | Para ServiceAccounts, OIDC, etc. |
| Autenticación básica | `username` + `password` | Mutuamente excluyente con `token` |
| Certificado mTLS | `client-certificate` + `client-key` | Combinable con token |

- Se gestiona con: `kubectl config set-credentials`

### Bloque `contexts`

Un contexto es un **perfil de conexión**: une un cluster, un usuario y un namespace bajo un nombre.

```yaml
contexts:
  - name: context-desarrollo
    context:
      cluster: produccion          # referencia a clusters[].name
      user: desarrollo1            # referencia a users[].name
      namespace: trabajo           # namespace por defecto para este contexto
```

- Cada campo (`cluster`, `user`, `namespace`) es **opcional**; los que falten usan el valor por defecto del sistema
- El mismo cluster puede tener múltiples contextos (ej: uno por namespace o por rol)
- Se gestiona con: `kubectl config set-context`

> **Importante:** `kubectl config set-context` **NO verifica** que el cluster, usuario o namespace existan en ningún cluster. Solo escribe en el fichero kubeconfig. El error aparecerá al intentar conectarse con ese contexto.

### `current-context`

Indica qué contexto usa `kubectl` cuando no se especifica ninguno.

```yaml
current-context: context-desarrollo
```

- Se cambia con: `kubectl config use-context <nombre>`

---

## 03 — Kubeconfig en Minikube

Minikube configura automáticamente el kubeconfig al crear el cluster. Puedes ver los perfiles disponibles:

```bash
minikube profile list
```

Salida de ejemplo:
```
┌──────────┬────────┬─────────┬──────────────┬─────────┬─────────┬───────┬────────────────┬────────────────────┐
│ PROFILE  │ DRIVER │ RUNTIME │      IP      │ VERSION │ STATUS  │ NODES │ ACTIVE PROFILE │ ACTIVE KUBECONTEXT │
├──────────┼────────┼─────────┼──────────────┼─────────┼─────────┼───────┼────────────────┼────────────────────┤
│ minikube │ docker │ docker  │ 192.168.49.2 │ v1.35.0 │ Stopped │ 1     │ *              │ *                  │
└──────────┴────────┴─────────┴──────────────┴─────────┴─────────┴───────┴────────────────┴────────────────────┘
```

### Comandos de inspección

```bash
# Ver el fichero kubeconfig completo
kubectl config view

# Ver sólo el contexto activo en este momento
kubectl config current-context

# Listar todos los contextos disponibles
kubectl config get-contexts

# Listar todos los clusters registrados
kubectl config get-clusters

# Listar todos los usuarios registrados
kubectl config get-users
```

### Cambiar el contexto activo

```bash
kubectl config use-context <nombre-contexto>

# Ejemplo con minikube:
kubectl config use-context minikube
```

---

## 04 — Añadir y gestionar clusters

### Añadir un cluster al config por defecto

```bash
kubectl config set-cluster cluster2 --server=http://1.2.3.4
```

Cómo queda en `~/.kube/config` tras el comando:
```yaml
clusters:
  - name: cluster2
    cluster:
      server: http://1.2.3.4
```

### Añadir un cluster a un fichero alternativo

```bash
kubectl --kubeconfig=config_alternativo config set-cluster cluster2 --server=http://1.2.3.4
```

- Si `config_alternativo` **no existe**, kubectl lo **crea** automáticamente
- Si ya existe, añade o actualiza la entrada del cluster
- El flag `--kubeconfig` va siempre **antes** de `config`

El fichero `config_alternativo` nuevo tendrá esta estructura mínima:
```yaml
apiVersion: v1
kind: Config
clusters:
  - name: cluster2
    cluster:
      server: http://1.2.3.4
contexts: []
current-context: ""
preferences: {}
users: []
```

### Añadir un cluster con certificado CA

El flag `--certificate-authority` indica a kubectl el fichero de la CA del cluster para verificar el certificado TLS del API Server.

```bash
# Linux / Mac:
kubectl config set-cluster cluster2 \
  --server=https://1.2.3.4:6443 \
  --certificate-authority=/home/dev/certs/ca.crt

# Windows:
kubectl config set-cluster cluster2 `
  --server=https://1.2.3.4:6443 `
  --certificate-authority=C:\Users\jho_j\certs\ca.crt
```

Resultado en el kubeconfig:
```yaml
clusters:
  - name: cluster2
    cluster:
      server: https://1.2.3.4:6443
      certificate-authority: C:\Users\jho_j\certs\ca.crt
```

### Usar el nuevo cluster

Para conectarse al nuevo cluster necesitas un contexto que lo referencie (ver Sección 06). Una vez creado el contexto:

```bash
# Cambiar al contexto que usa cluster2
kubectl config use-context contexto-cluster2

# Verificar la conexión
kubectl cluster-info
kubectl get nodes
```

---

## 05 — Añadir usuarios y credenciales

### Usuario con username y password (autenticación básica HTTP)

```bash
kubectl config set-credentials usu1 --username=usu1 --password=miPassword
```

> La contraseña se almacena en **texto plano** en el fichero kubeconfig. kubectl la envía al API Server codificada como HTTP Basic Auth (`Authorization: Basic base64(usuario:password)`). Este método **no se recomienda en producción** porque las credenciales quedan expuestas en el fichero.

Cómo queda en el config:
```yaml
users:
  - name: usu1
    user:
      password: miPassword
      username: usu1
```

### Usuario con certificado de cliente (mTLS)

Este es el método **más seguro y más habitual en producción**. El cliente presenta un certificado firmado por la CA del cluster para demostrar su identidad. El API Server verifica que ese certificado fue firmado por una CA de confianza.

```bash
# Opción A: dos comandos separados
kubectl config set-credentials usu2 \
  --client-certificate=C:\Users\jho_j\certs\cert.crt

kubectl config set-credentials usu2 \
  --client-key=C:\Users\jho_j\certs\cert.key

# Opción B: todo en un solo comando (recomendado)
kubectl config set-credentials usu2 \
  --client-certificate=C:\Users\jho_j\certs\cert.crt \
  --client-key=C:\Users\jho_j\certs\cert.key
```

> Si ejecutas los dos comandos por separado, el segundo simplemente **actualiza** la misma entrada `usu2` añadiendo el campo que faltaba.

Cómo queda en el config:
```yaml
users:
  - name: usu2
    user:
      client-certificate: C:\Users\jho_j\certs\cert.crt
      client-key: C:\Users\jho_j\certs\cert.key
```

**¿Para qué sirve cada tipo?**

| Tipo | Cuándo usarlo |
|------|--------------|
| `username` + `password` | Entornos de desarrollo o laboratorio. Rápido pero inseguro. |
| `client-certificate` + `client-key` | Producción. El certificado identifica al usuario criptográficamente. Se puede revocar a nivel de CA. |
| `token` | Para ServiceAccounts dentro del cluster o flujos OIDC (SSO). |

---

## 06 — Crear y gestionar contextos

### Crear un contexto en el config por defecto

```bash
kubectl config set-context contexto-cluster2 \
  --cluster=cluster2 \
  --user=usu1 \
  --namespace=desarrollo
```

> **Advertencia:** Este comando **solo escribe en el fichero kubeconfig**. No verifica que `cluster2`, `usu1` o el namespace `desarrollo` existan en ningún cluster real. Si alguno no existe, el error aparecerá al intentar usar el contexto con `kubectl get pods`, por ejemplo.

Cómo queda en el config:
```yaml
contexts:
  - name: contexto-cluster2
    context:
      cluster: cluster2
      namespace: desarrollo
      user: usu1
```

### Crear un contexto en un fichero alternativo

```bash
kubectl --kubeconfig=config_alternativo config set-context contexto-cluster2 \
  --cluster=cluster2 \
  --user=usu1 \
  --namespace=desarrollo
```

Este comando escribe el contexto en `config_alternativo` en lugar del config por defecto. La lógica es la misma: `--kubeconfig` siempre antes de `config`.

### Activar el contexto en el fichero alternativo

```bash
kubectl --kubeconfig=config_alternativo config use-context contexto-cluster2
```

Esto cambia el campo `current-context` **dentro de `config_alternativo`**. No afecta para nada al `~/.kube/config` principal.

### Usar el fichero alternativo para lanzar comandos

```bash
# Flag por comando:
kubectl --kubeconfig=config_alternativo get pods
kubectl --kubeconfig=config_alternativo get nodes

# O activarlo para toda la sesión del terminal:

# Windows PowerShell:
$env:KUBECONFIG = "config_alternativo"
kubectl get pods    # ya usa config_alternativo

# Linux / Mac:
export KUBECONFIG=config_alternativo
kubectl get pods
```

---

## Ejemplo completo: construir una configuración desde cero

Secuencia completa que replica el ejemplo de la documentación oficial:

```bash
# 1. Crear el usuario con autenticación básica
kubectl config set-credentials myself --username=admin --password=secret

# 2. Añadir el cluster
kubectl config set-cluster local-server --server=http://localhost:8080

# 3. Crear el contexto que une cluster + usuario
kubectl config set-context default-context --cluster=local-server --user=myself

# 4. Activar el contexto
kubectl config use-context default-context

# 5. Fijar el namespace por defecto del contexto
kubectl config set contexts.default-context.namespace the-right-prefix

# 6. Ver el resultado completo
kubectl config view
```

Resultado esperado de `kubectl config view`:
```yaml
apiVersion: v1
kind: Config
current-context: default-context
preferences: {}
clusters:
  - name: local-server
    cluster:
      server: http://localhost:8080
users:
  - name: myself
    user:
      password: secret
      username: admin
contexts:
  - name: default-context
    context:
      cluster: local-server
      namespace: the-right-prefix
      user: myself
```

---

## Referencia rápida de comandos

```bash
# --- Inspección ---
kubectl config view                         # ver todo el kubeconfig
kubectl config current-context             # contexto activo
kubectl config get-contexts                # listar contextos
kubectl config get-clusters                # listar clusters
kubectl config get-users                   # listar usuarios

# --- Clusters ---
kubectl config set-cluster <nombre> --server=<url>
kubectl config set-cluster <nombre> --server=<url> --certificate-authority=<ruta>
kubectl config set-cluster <nombre> --insecure-skip-tls-verify=true

# --- Usuarios ---
kubectl config set-credentials <nombre> --username=<user> --password=<pass>
kubectl config set-credentials <nombre> --token=<token>
kubectl config set-credentials <nombre> --client-certificate=<ruta> --client-key=<ruta>

# --- Contextos ---
kubectl config set-context <nombre> --cluster=<cluster> --user=<usuario> --namespace=<ns>
kubectl config use-context <nombre>                       # activar contexto
kubectl config delete-context <nombre>                    # eliminar contexto

# --- Fichero alternativo (el flag --kubeconfig siempre antes de config) ---
kubectl --kubeconfig=<fichero> config view
kubectl --kubeconfig=<fichero> config set-cluster <nombre> --server=<url>
kubectl --kubeconfig=<fichero> config set-credentials <nombre> --username=x --password=y
kubectl --kubeconfig=<fichero> config set-context <nombre> --cluster=x --user=y
kubectl --kubeconfig=<fichero> config use-context <nombre>

# --- Variable de entorno (aplica a toda la sesión) ---
export KUBECONFIG=<ruta>          # Linux/Mac
$env:KUBECONFIG = "<ruta>"        # Windows PowerShell
```
