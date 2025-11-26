# Práctica 2: Despliegue de aplicaciones usando Kubernetes/Docker
## Sistemas Distribuidos - Curso 2024/2025 - Programacion de Sistemas Distribuidos
## Alejandro Mamán López-Mingo 
---

## 📋 Índice

1. [Introducción](#introducción)
2. [Parte 1: Configuración Básica del Cluster](#parte-1-configuración-básica-del-cluster)
3. [Parte 2: Despliegue Básico con Docker y Kubernetes](#parte-2-despliegue-básico-con-docker-y-kubernetes)
4. [Parte 3: Configuración Avanzada con NFS](#parte-3-configuración-avanzada-con-nfs)
5. [Parte 4: Pruebas y Verificación](#parte-4-pruebas-y-verificación)
6. [Resumen de Problemas y Soluciones](#resumen-de-problemas-y-soluciones)
7. [Referencias y Comandos Útiles](#referencias-y-comandos-útiles)

---

## 🎯 Introducción

Esta guía documenta el proceso completo de despliegue de un sistema distribuido de gestión de archivos en Kubernetes, desde la configuración básica hasta una implementación avanzada con almacenamiento compartido NFS y alta disponibilidad.

### Objetivo del Proyecto

Implementar un sistema distribuido con tres componentes:
- **Broker:** Coordina las conexiones entre clientes y servidores
- **Servidores:** Gestionan archivos remotos (listado, subida, descarga)
- **Cliente:** Interfaz para interactuar con el sistema

### Alcance de esta Guía

Esta documentación cubre dos niveles de implementación:

**Nivel 1 - Configuración Básica (Aprobado):**
- Cluster Kubernetes de 3 nodos
- 1 réplica del broker y 1 del servidor
- Servicios NodePort para acceso externo

**Nivel 2 - Configuración Avanzada (Nota 10):**
- 3 réplicas del servidor con alta disponibilidad
- Almacenamiento compartido mediante NFS
- Persistencia de datos entre pods

### Requisitos Previos

- 3 instancias EC2 en AWS (Ubuntu 22.04)
- Docker instalado en todas las instancias
- Cuenta en Docker Hub
- Conocimientos básicos de Kubernetes y Docker
- Acceso SSH a las instancias

---

## 🏗️ Arquitectura Final Implementada

```
┌──────────────────────────────────────────────────────────────────┐
│  k8smaster0.psdi.org (control-plane + NFS Server)               │
│  IP: 172.31.64.84                                               │
│  Rol: Master del cluster + Servidor NFS                         │
│  NFS: /mnt/nfs-filemanager (almacenamiento compartido)          │
└──────────────────────────────────────────────────────────────────┘
                            │
                ┌───────────┴───────────┐
                │                       │
┌───────────────▼───────────┐  ┌───────▼──────────────────────────┐
│ k8sslave1.psdi.org         │  │ k8sslave2.psdi.org               │
│ IP: 172.31.31.30           │  │ IP: 172.31.72.209                │
│ Label: node-role=broker    │  │ Label: node-role=server          │
│                            │  │ NFS Client: instalado            │
│ ┌────────────────────────┐ │  │ ┌──────────────────────────────┐ │
│ │ broker-deployment      │ │  │ │ server-deployment (3 pods)   │ │
│ │ 1 réplica              │ │  │ │ - server-xxx-7vpf9           │ │
│ │ Puerto: 32002          │ │  │ │ - server-xxx-cvltl           │ │
│ └────────────────────────┘ │  │ │ - server-xxx-q9j**           │ │
│                            │  │ │ Volumen NFS: /FileManagerDir │ │
│                            │  │ └──────────────────────────────┘ │
└────────────────────────────┘  └──────────────────────────────────┘
        │                                    │
        │         ┌──────────────────────────┘
        │         │
        ▼         ▼
┌──────────────────────────────────┐
│  Cliente (clientFileManager)     │
│  Conecta a: 172.31.31.30:32002  │
│  Balanceo automático entre pods  │
└──────────────────────────────────┘
```

### Componentes Desplegados

| Componente | Nodo | Réplicas | Puerto | Imagen |
|------------|------|----------|--------|--------|
| Broker | k8sslave1 | 1 | 32002 | d1n0s/kubernetes-practica2broker:v1 |
| Server | k8sslave2 | 3 | 32001 | d1n0s/kubernetes-practica2server:v2 |
| NFS Server | k8smaster0 | - | 2049 | Sistema (nfs-kernel-server) |

### Recursos de Kubernetes

| Recurso | Nombre | Descripción |
|---------|--------|-------------|
| PersistentVolume | server-pv-nfs | Volumen NFS de 5Gi |
| PersistentVolumeClaim | server-pvc-nfs | Claim vinculado al PV |
| Service | brokerservice | NodePort 32002 |
| Service | serverservice | NodePort 32001 |

---

## 📘 Parte 1: Configuración Básica del Cluster

### 1.1 Añadir Nodos Worker al Cluster

**Generar token de join en k8smaster0:**

```bash
# Crear script para añadir nodos
cd ~/kub
./kub_addNode.sh 172.31.31.30   # IP de k8sslave1
./kub_addNode.sh 172.31.72.209  # IP de k8sslave2
```

**Verificar que los nodos se unieron correctamente:**

```bash
kubectl get nodes -o wide
```

**Salida esperada:**
```
NAME                  STATUS   ROLES           VERSION
k8smaster0.psdi.org   Ready    control-plane   v1.28.15
k8sslave1.psdi.org    Ready    worker          v1.28.15
k8sslave2.psdi.org    Ready    worker          v1.28.15
```

### 1.2 Etiquetar los Nodos

Las etiquetas permiten usar `nodeSelector` para controlar dónde se ejecutan los pods:

```bash
# Etiquetar k8sslave1 para el broker
kubectl label nodes k8sslave1.psdi.org node-role=broker

# Etiquetar k8sslave2 para los servidores
kubectl label nodes k8sslave2.psdi.org node-role=server

# Verificar las etiquetas
kubectl get nodes --show-labels
```

---

## 🐳 Parte 2: Despliegue Básico con Docker y Kubernetes

### 2.1 Crear Imágenes Docker

#### Dockerfile del Broker

**Archivo:** `DockerfileBroker`

```dockerfile
FROM ubuntu:20.04
RUN apt-get update
RUN apt-get install -y software-properties-common curl
EXPOSE 32002
COPY brokerFileManager /
RUN chmod +x /brokerFileManager
CMD /brokerFileManager
```

**Construir y subir:**

```bash
cd ~/Practica2/BROKER

# Construir la imagen
docker build -t d1n0s/kubernetes-practica2broker:v1 -f DockerfileBroker .

# Login en Docker Hub
docker login

# Subir la imagen
docker push d1n0s/kubernetes-practica2broker:v1
```

#### Dockerfile del Servidor

**Archivo:** `DockerfileServer`

```dockerfile
FROM ubuntu:20.04
RUN apt-get update
RUN apt-get install -y software-properties-common curl
EXPOSE 32001
COPY serverFileManager /
RUN chmod +x /serverFileManager
RUN mkdir FileManagerDir
COPY resolv.conf /
CMD cp resolv.conf /etc/resolv.conf && /serverFileManager 172.31.31.30 32002 $(curl -s https://api.ipify.org) 32001
```

**⚠️ Nota Importante:** El servidor se conecta al broker usando la IP privada `172.31.31.30` (k8sslave1). Esto es un requisito del profesor.

**Construir y subir:**

```bash
cd ~/Practica2/SERVER

# Construir la imagen (versión 2 para evitar problemas de caché)
docker build -t d1n0s/kubernetes-practica2server:v2 -f DockerfileServer .

# Subir la imagen
docker push d1n0s/kubernetes-practica2server:v2
```

**💡 Tip:** Si necesitas actualizar una imagen, cambia el tag (v2 → v3) para forzar a Kubernetes a descargar la nueva versión.

### 2.2 Crear Deployments de Kubernetes

**⚠️ Consideración Importante sobre Nomenclatura:**

Kubernetes requiere que todos los nombres de recursos sigan la convención RFC 1123:
- Solo letras minúsculas, números y guiones
- ❌ Incorrecto: `BrokerDeployment`, `ServerDeployment`
- ✅ Correcto: `broker-deployment`, `server-deployment`

#### Deployment del Broker

**Archivo:** `DeploymentBroker.yml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
 name: broker-deployment  # ⚠️ Nombre en minúsculas
 namespace: default
spec:
 replicas: 1
 selector:
  matchLabels:
   app: brokerfilemanager
 template:
  metadata:
   labels:
    app: brokerfilemanager
  spec:
   nodeSelector:
    node-role: broker  # Se ejecutará en k8sslave1
   containers:
   - name: broker-deployment
     image: docker.io/d1n0s/kubernetes-practica2broker:v1
     ports:
     - containerPort: 32002
```

**Aplicar el deployment:**

```bash
cd ~/Practica2/BROKER
kubectl apply -f DeploymentBroker.yml

# Verificar que se creó correctamente
kubectl get deployment broker-deployment
kubectl get pods -l app=brokerfilemanager
```

#### Deployment del Servidor (Básico - 1 réplica)

**Archivo:** `DeploymentServer.yml` (versión inicial)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
 name: server-deployment  # ⚠️ Nombre en minúsculas
 namespace: default
spec:
 replicas: 1  # Por ahora solo 1 réplica
 selector:
  matchLabels:
   app: server-deploy  # ⚠️ Debe coincidir con el Service
 template:
  metadata:
   labels:
    app: server-deploy
  spec:
   nodeSelector:
    node-role: server  # Se ejecutará en k8sslave2
   containers:
   - name: server-deployment
     image: docker.io/d1n0s/kubernetes-practica2server:v2
     ports:
     - containerPort: 32001
```

**Aplicar el deployment:**

```bash
cd ~/Practica2/SERVER
kubectl apply -f DeploymentServer.yml

# Verificar
kubectl get deployment server-deployment
kubectl get pods -l app=server-deploy -o wide
```

### 2.3 Crear Services para Exponer los Pods

Los Services permiten acceder a los pods desde fuera del cluster usando NodePort.

#### Service del Broker

**Archivo:** `ServiceBroker.yml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: brokerservice
  namespace: default
spec:
  type: NodePort
  ports:
  - port: 32002
    targetPort: 32002
    nodePort: 32002
  selector:
    app: brokerfilemanager  # ⚠️ Debe coincidir con el Deployment
```

**Aplicar:**

```bash
kubectl apply -f ServiceBroker.yml
kubectl get svc brokerservice
```

#### Service del Servidor

**Archivo:** `ServiceServer.yml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: serverservice
  namespace: default
spec:
  type: NodePort
  ports:
  - port: 32001
    targetPort: 32001
    nodePort: 32001
  selector:
    app: server-deploy  # ⚠️ Debe coincidir con el Deployment
```

**Aplicar:**

```bash
kubectl apply -f ServiceServer.yml
kubectl get svc serverservice
```

### 2.4 Configurar Security Groups en AWS

Para acceder a los servicios desde fuera, debes abrir los puertos en AWS:

1. Ve a **AWS Console → EC2 → Security Groups**
2. Selecciona el security group de tus instancias
3. **Editar reglas de entrada** y añadir:

| Tipo | Puerto | Origen | Descripción |
|------|--------|--------|-------------|
| Custom TCP | 32002 | 0.0.0.0/0 | Broker NodePort |
| Custom TCP | 32001 | 0.0.0.0/0 | Server NodePort |

### 2.5 Probar la Configuración Básica

**Verificar que todos los pods estén corriendo:**

```bash
kubectl get pods -o wide
```

**Salida esperada:**
```
NAME                                 READY   STATUS    NODE
broker-deployment-6fd556654c-jzdsx   1/1     Running   k8sslave1.psdi.org
server-deployment-689b756d6-6jqz8    1/1     Running   k8sslave2.psdi.org
```

**Probar el cliente:**

```bash
cd ~/Practica2
./clientFileManager 172.31.31.30 32002

# Comandos disponibles:
# - ls: Lista archivos locales
# - lls: Lista archivos remotos en el servidor
# - upload archivo.txt: Sube archivo
# - download archivo.txt: Descarga archivo
```

**✅ CONFIGURACIÓN BÁSICA COMPLETADA** - Con esto tienes aprobada la práctica.

---

## 🚀 Parte 3: Configuración Avanzada con NFS

Esta sección implementa almacenamiento compartido NFS para alcanzar la **máxima calificación (10)**. Permite tener 3 réplicas del servidor compartiendo los mismos archivos.

### ¿Por qué NFS?

- ✅ **Alta disponibilidad:** 3 réplicas del servidor corriendo simultáneamente
- ✅ **Persistencia:** Los archivos se mantienen aunque un pod se reinicie
- ✅ **Compartición:** Todas las réplicas ven los mismos archivos en tiempo real
- ✅ **Escalabilidad:** Fácil añadir más réplicas sin perder datos

### 3.1 Instalar y Configurar Servidor NFS

**En k8smaster0:**

```bash
# Actualizar repositorios
sudo apt-get update

# Instalar servidor NFS
sudo apt-get install -y nfs-kernel-server

# Crear directorio compartido
sudo mkdir -p /mnt/nfs-filemanager

# Configurar permisos
sudo chown nobody:nogroup /mnt/nfs-filemanager
sudo chmod 777 /mnt/nfs-filemanager
```

**Configurar exportación NFS:**

```bash
# Añadir al archivo de exportaciones
echo "/mnt/nfs-filemanager *(rw,sync,no_subtree_check,no_root_squash)" | sudo tee -a /etc/exports

# Aplicar configuración
sudo exportfs -ra

# Verificar que el export está activo
sudo exportfs -v
```

**Salida esperada:**
```
/mnt/nfs-filemanager
        <world>(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,no_root_squash,no_all_squash)
```

**Verificar servicio:**

```bash
sudo systemctl status nfs-kernel-server
```

**⚠️ Problema Común 1: Ruta sin barra inicial**

Si ves el error `exportfs: Failed to stat mnt/nfs-filemanager: No such file or directory`:

```bash
# Eliminar línea incorrecta
sudo sed -i '/^mnt\/nfs-filemanager/d' /etc/exports

# Añadir correctamente (con / inicial)
echo "/mnt/nfs-filemanager *(rw,sync,no_subtree_check,no_root_squash)" | sudo tee -a /etc/exports

# Aplicar
sudo exportfs -ra
```

### 3.2 Instalar Cliente NFS en Workers

**En k8sslave2 (usando kubectl debug desde k8smaster0):**

```bash
# Entrar al nodo con chroot
kubectl debug node/k8sslave2.psdi.org -it --image=ubuntu -- chroot /host bash

# Dentro del nodo, instalar cliente NFS
apt-get update
apt-get install -y nfs-common

# Salir
exit
```

**Verificar instalación:**

```bash
kubectl debug node/k8sslave2.psdi.org -it --image=ubuntu -- chroot /host bash -c "dpkg -l | grep nfs-common"
```

**Salida esperada:**
```
ii  nfs-common  1:2.6.1-1ubuntu1.2  amd64  NFS support files common to client and server
```

**💡 Nota:** Los pods de debug temporales pueden eliminarse después:

```bash
kubectl delete pod -l app=node-debugger
```

### 3.3 Configurar Security Groups de AWS para NFS

Antes de crear los recursos de Kubernetes, debes abrir los puertos NFS en AWS:

1. Ve a **AWS Console → EC2 → Security Groups**
2. Selecciona el security group de **k8smaster0**
3. **Editar reglas de entrada** y añadir:

| Tipo | Protocolo | Puerto | Origen | Descripción |
|------|-----------|--------|--------|-------------|
| TCP personalizado | TCP | 2049 | 172.31.0.0/16 | NFS |
| TCP personalizado | TCP | 111 | 172.31.0.0/16 | RPC (portmapper) |

**⚠️ Problema Común 2: Connection timed out al montar NFS**

Si los pods muestran `mount.nfs: Connection timed out`, es porque el Security Group bloquea los puertos NFS. Asegúrate de añadir las reglas anteriores.

### 3.4 Crear PersistentVolume NFS

**Archivo:** `pv-nfs.yml`



```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: server-pv-nfs
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteMany  # Permite que múltiples pods lo usen simultáneamente
  nfs:
    server: 172.31.64.84  # IP privada de k8smaster0
    path: /mnt/nfs-filemanager
  storageClassName: nfs
```

**Crear el archivo y aplicarlo:**

```bash
cd ~/Practica2/SERVIDOR_NFS

# Crear el archivo pv-nfs.yml con el contenido anterior
nano pv-nfs.yml

# Aplicar
kubectl apply -f pv-nfs.yml

# Verificar
kubectl get pv
```

**Salida esperada:**
```
NAME            CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM
server-pv-nfs   5Gi        RWX            Retain           Available
```

### 3.5 Crear PersistentVolumeClaim NFS

**Archivo:** `pvc-nfs.yml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: server-pvc-nfs
spec:
  accessModes:
    - ReadWriteMany  # Debe coincidir con el PV
  resources:
    requests:
      storage: 5Gi  # Solicita 5Gi
  storageClassName: nfs  # Debe coincidir con el PV
```

**Aplicar:**

```bash
# Crear el archivo
nano pvc-nfs.yml

# Aplicar
kubectl apply -f pvc-nfs.yml

# Verificar que se vinculó al PV
kubectl get pvc
kubectl get pv
```

**Salida esperada:**
```
NAME             STATUS   VOLUME          CAPACITY   ACCESS MODES
server-pvc-nfs   Bound    server-pv-nfs   5Gi        RWX
```

**Estado `Bound` significa que el PVC encontró el PV y está listo para usar.**

### 3.6 Actualizar Deployment del Servidor con NFS

Ahora modifica el `DeploymentServer.yml` para usar el volumen NFS y escalar a 3 réplicas:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
 name: server-deployment
 namespace: default
spec:
 replicas: 3  # ← CAMBIO: Escalar de 1 a 3 réplicas
 selector:
  matchLabels:
   app: server-deploy
 template:
  metadata:
   labels:
    app: server-deploy
  spec:
   nodeSelector:
    node-role: server
   containers:
   - name: server-deployment
     image: docker.io/d1n0s/kubernetes-practica2server:v2
     ports:
     - containerPort: 32001
     volumeMounts:  # ← NUEVO: Montar el volumen NFS
     - name: filemanager-storage-nfs
       mountPath: /FileManagerDir  # Donde la app guarda archivos
   volumes:  # ← NUEVO: Definir el volumen desde el PVC
   - name: filemanager-storage-nfs
     persistentVolumeClaim:
       claimName: server-pvc-nfs  # Referencia al PVC creado
```

**Aplicar los cambios:**

```bash
# Eliminar el deployment anterior
kubectl delete deployment server-deployment

# Aplicar el nuevo con NFS
kubectl apply -f DeploymentServer.yml

# Observar cómo se crean las 3 réplicas
kubectl get pods -w
```

**⚠️ Problema Común 3: ImagePullBackOff**

Si los pods muestran `ImagePullBackOff`, verifica la versión de la imagen:

```bash
# Ver el error
kubectl describe pod server-deployment-xxx

# Si dice que no encuentra v3, verifica el deployment
kubectl get deployment server-deployment -o yaml | grep image

# Debe ser v2 (que existe en Docker Hub)
# Si está mal, edita DeploymentServer.yml y vuelve a aplicar
```

**⚠️ Problema Común 4: MountVolume.SetUp failed - Connection timed out**

Este es el problema más común al configurar NFS. Los pods quedan en `ContainerCreating`:

```bash
# Ver el error
kubectl describe pod server-deployment-xxx
```

**Causa:** Security Group de AWS bloqueando puertos NFS.

**Solución:** Añadir reglas en Security Group (ver sección 3.3).

**Salida esperada cuando todo funciona:**
```
NAME                                 READY   STATUS    NODE
broker-deployment-6fd556654c-jzdsx   1/1     Running   k8sslave1.psdi.org
server-deployment-6bc5f558c5-7vpf9   1/1     Running   k8sslave2.psdi.org
server-deployment-6bc5f558c5-cvltl   1/1     Running   k8sslave2.psdi.org
server-deployment-6bc5f558c5-q9j**   1/1     Running   k8sslave2.psdi.org
```

---

## ✅ Parte 4: Pruebas y Verificación

Esta sección documenta las pruebas realizadas para verificar que el sistema funciona correctamente con NFS.

### 4.1 Verificar Estado del Cluster

```bash
# Ver todos los pods
kubectl get pods -o wide

# Ver servicios
kubectl get svc

# Ver volúmenes
kubectl get pv,pvc
```

### 4.2 Prueba de Persistencia de Archivos

**Paso 1: Crear un archivo de prueba**

```bash
cd ~/Practica2
echo "Esto es una prueba" > Prueba.txt
cat Prueba.txt  # Verificar contenido
```

**Paso 2: Subir archivo al sistema**

```bash
# Conectar al broker
./clientFileManager 172.31.31.30 32002

# Dentro del cliente, subir el archivo
upload Prueba.txt

# Verificar que se subió
lls
```

**Salida esperada:**
```
Enter command:
upload Prueba.txt
Coping file Prueba.txt in to the FileManager path
Reading file: Prueba.txt 19 bytes

Enter command:
lls
Listing files fileManager path
FileManagerDir/Prueba.txt
```

**Paso 3: Salir y reconectar (puede conectar a otra réplica)**

```bash
# Salir (Ctrl+C si "exit" no funciona)
# Volver a conectar
./clientFileManager 172.31.31.30 32002

# Listar archivos
lls
```

**Resultado esperado:** El archivo `Prueba.txt` debe seguir ahí, confirmando la persistencia.

### 4.3 Verificar Archivos en el Servidor NFS

**En k8smaster0:**

```bash
# Ver archivos en el directorio NFS
ls -la /mnt/nfs-filemanager/

# Ver contenido del archivo
cat /mnt/nfs-filemanager/Prueba.txt
```

**Salida esperada:**
```
total 12
drwxrwxrwx 2 nobody nogroup 4096 Nov 26 17:45 .
drwxr-xr-x 3 root   root    4096 Nov 26 17:10 ..
-rw-r--r-- 1 nobody nogroup   19 Nov 26 17:45 Prueba.txt
```

### 4.4 Verificar Logs de las Réplicas

```bash
# Obtener nombres de los pods
kubectl get pods | grep server-deployment

# Ver logs de cada réplica (reemplaza con tus nombres reales)
kubectl logs server-deployment-6bc5f558c5-7vpf9
kubectl logs server-deployment-6bc5f558c5-cvltl
kubectl logs server-deployment-6bc5f558c5-q9j**
```

Deberías ver que todas las réplicas están registradas en el broker y listas para recibir conexiones.

### 4.5 Prueba de Alta Disponibilidad (Opcional)

**Simular fallo de un pod:**

```bash
# Eliminar un pod
kubectl delete pod server-deployment-6bc5f558c5-7vpf9

# Kubernetes creará uno nuevo automáticamente
kubectl get pods -w

# El archivo debe seguir accesible
./clientFileManager 172.31.31.30 32002
lls  # Prueba.txt sigue ahí
```

**✅ Resultados obtenidos en esta práctica:**
- ✅ Archivo `Prueba.txt` subido correctamente
- ✅ Archivo persiste entre diferentes conexiones
- ✅ Las 3 réplicas ven el mismo archivo
- ✅ El archivo está físicamente en `/mnt/nfs-filemanager/` del servidor NFS
- ✅ Alta disponibilidad confirmada: si un pod muere, los datos siguen disponibles

**🎉 CONFIGURACIÓN COMPLETA - SISTEMA FUNCIONANDO AL 100%**

---

## 🔥 Resumen de Problemas y Soluciones

Durante la implementación se encontraron los siguientes problemas. Esta sección te ayudará si encuentras errores similares:

### Problema 1: Nombres de Recursos en Mayúsculas

**Error:**
```
The Deployment "BrokerDeployment" is invalid: metadata.name: Invalid value: "BrokerDeployment": a lowercase RFC 1123 subdomain must consist of lower case alphanumeric characters, '-' or '.', and must start and end with an alphanumeric character
```

**Causa:** Kubernetes requiere nombres en minúsculas según RFC 1123.

**Solución:**
- ❌ `BrokerDeployment` → ✅ `broker-deployment`
- ❌ `ServerDeployment` → ✅ `server-deployment`

### Problema 2: Selector Mismatch entre Service y Deployment

**Error:** Service no encuentra los pods, `kubectl get endpoints` muestra vacío.

**Causa:** El selector del Service no coincide con las labels del Deployment.

**Solución:**
```yaml
# Service
selector:
  app: server-deploy  # ← Debe coincidir

# Deployment
template:
  metadata:
    labels:
      app: server-deploy  # ← Debe coincidir
```

### Problema 3: Ruta NFS sin Barra Inicial

**Error:**
```
exportfs: Failed to stat mnt/nfs-filemanager: No such file or directory
```

**Causa:** Falta `/` al inicio de la ruta en `/etc/exports`.

**Solución:**
```bash
# Incorrecto
echo "mnt/nfs-filemanager *(rw,sync,no_subtree_check,no_root_squash)" | sudo tee -a /etc/exports

# Correcto
echo "/mnt/nfs-filemanager *(rw,sync,no_subtree_check,no_root_squash)" | sudo tee -a /etc/exports

# Corregir si ya está mal
sudo sed -i '/^mnt\/nfs-filemanager/d' /etc/exports
echo "/mnt/nfs-filemanager *(rw,sync,no_subtree_check,no_root_squash)" | sudo tee -a /etc/exports
sudo exportfs -ra
```

### Problema 4: Mount Timeout - Connection Timed Out

**Error en pod:**
```
MountVolume.SetUp failed for volume "server-pv-nfs": mount failed: exit status 32
Mounting command: mount
Output: mount.nfs: Connection timed out
```

**Causa:** Security Group de AWS bloqueando puertos NFS (2049 y 111).

**Solución:**

1. Ve a AWS Console → EC2 → Security Groups
2. Selecciona el security group de **k8smaster0**
3. Añade reglas de entrada:
   - Puerto 2049 TCP desde 172.31.0.0/16 (NFS)
   - Puerto 111 TCP desde 172.31.0.0/16 (RPC)

### Problema 5: ImagePullBackOff

**Error:**
```
Failed to pull image "docker.io/d1n0s/kubernetes-practica2server:v3": rpc error: code = NotFound desc = failed to pull and unpack image
```

**Causa:** Tag de imagen no existe en Docker Hub.

**Solución:**
```bash
# Verificar qué imagen necesitas
docker images | grep kubernetes-practica2server

# Corregir en DeploymentServer.yml
image: docker.io/d1n0s/kubernetes-practica2server:v2  # ← v2 que sí existe

# Aplicar
kubectl delete deployment server-deployment
kubectl apply -f DeploymentServer.yml
```

### Problema 6: PVC en Estado Pending

**Error:** `kubectl get pvc` muestra `STATUS: Pending`.

**Causa:** No hay PV disponible que coincida con las especificaciones del PVC.

**Solución:**
```bash
# Verificar que PV y PVC tienen los mismos:
# - storageClassName
# - accessModes
# - capacity (PV >= PVC)

kubectl describe pvc server-pvc-nfs  # Ver por qué está pending
kubectl get pv  # Verificar PV disponibles
```

---

## 📚 Referencias y Comandos Útiles

### Comandos de Kubernetes Más Usados

**Visualización:**
```bash
# Ver todo el estado del cluster
kubectl get all -o wide

# Ver nodos con detalles
kubectl get nodes -o wide

# Ver pods de un deployment específico
kubectl get pods -l app=server-deploy

# Ver eventos recientes
kubectl get events --sort-by='.lastTimestamp' | head -20

# Ver logs de un pod
kubectl logs <nombre-pod>
kubectl logs <nombre-pod> --follow  # En tiempo real
```

**Gestión de Recursos:**
```bash
# Aplicar configuración
kubectl apply -f <archivo.yml>

# Eliminar recurso
kubectl delete deployment <nombre>
kubectl delete pod <nombre>

# Escalar deployment
kubectl scale deployment server-deployment --replicas=5

# Ver detalles de un recurso
kubectl describe pod <nombre-pod>
kubectl describe deployment <nombre>
kubectl describe pvc <nombre>
```

**Depuración:**
```bash
# Entrar a un pod
kubectl exec -it <nombre-pod> -- /bin/bash

# Ver configuración en YAML
kubectl get deployment server-deployment -o yaml

# Ver qué imagen está usando un pod
kubectl get pod <nombre-pod> -o jsonpath='{.spec.containers[0].image}'
```

### Comandos de Docker

```bash
# Construir y subir imágenes
docker build -t usuario/imagen:tag -f Dockerfile .
docker push usuario/imagen:tag

# Ver imágenes locales
docker images

# Eliminar imagen
docker rmi usuario/imagen:tag

# Login en Docker Hub
docker login
```

### Comandos de NFS

```bash
# Ver exports activos
sudo exportfs -v

# Recargar configuración
sudo exportfs -ra

# Estado del servicio
sudo systemctl status nfs-kernel-server

# Ver montajes NFS en el sistema
mount | grep nfs
```

### Referencias

- [Documentación oficial de Kubernetes](https://kubernetes.io/docs/home/)
- [Docker Hub](https://hub.docker.com/)
- [NFS en Kubernetes](https://kubernetes.io/docs/concepts/storage/volumes/#nfs)
- [Calico CNI](https://docs.projectcalico.org/)

---

## ✅ Checklist de Progreso

### Parte 1: Configuración Básica del Cluster (Nota: 5)
- [x] Cluster Kubernetes inicializado con kubeadm
- [x] 3 nodos agregados y en estado Ready
- [x] Nodos etiquetados correctamente (broker, server)
- [x] Calico CNI instalado y funcionando

### Parte 2: Despliegue Básico (Nota: Aprobado)
- [x] Dockerfile del broker creado y construido
- [x] Dockerfile del servidor creado y construido
- [x] Imágenes subidas a Docker Hub (v1 broker, v2 server)
- [x] Deployment del broker aplicado y funcionando
- [x] Deployment del servidor aplicado (1 réplica)
- [x] Services NodePort creados (32002, 32001)
- [x] Security Groups de AWS configurados
- [x] Cliente se conecta y puede subir/listar archivos

### Parte 3: Configuración Avanzada con NFS (Nota: 10)
- [x] Servidor NFS instalado en k8smaster0
- [x] Directorio NFS configurado y exportado
- [x] Cliente NFS instalado en k8sslave2
- [x] Security Groups AWS con puertos NFS abiertos
- [x] PersistentVolume NFS creado y disponible
- [x] PersistentVolumeClaim NFS vinculado (Bound)
- [x] Deployment actualizado con volumen NFS
- [x] 3 réplicas del servidor corriendo
- [x] Todas las réplicas usando el mismo almacenamiento

### Parte 4: Pruebas y Verificación
- [x] Archivo subido a través del cliente
- [x] Persistencia verificada en múltiples conexiones
- [x] Archivo visible en el servidor NFS
- [x] Logs de las 3 réplicas verificados
- [x] Alta disponibilidad confirmada

### Problemas Resueltos
- [x] Nombres en mayúsculas → minúsculas
- [x] Selector mismatch corregido
- [x] Ruta NFS sin `/` inicial corregida
- [x] Security Groups AWS configurados para NFS
- [x] ImagePullBackOff resuelto (v3 → v2)

---

## 🎉 Resultado Final

**Estado del Sistema:**
- ✅ Cluster Kubernetes: 3 nodos operativos
- ✅ Broker: 1 réplica en k8sslave1
- ✅ Servidor: 3 réplicas en k8sslave2 compartiendo NFS
- ✅ Almacenamiento: NFS de 5Gi en k8smaster0
- ✅ Cliente: Conectando y operando correctamente
- ✅ Persistencia: Archivos compartidos entre todas las réplicas

**Calificación Esperada: 10/10**

**Tiempo Total de Implementación:** ~6 horas (incluyendo resolución de problemas)

---

**Autor:** Alejandro Mamán López-Mingo  
**Asignatura:** Programacion de Sistemas Distribuidos  
**Curso:** 2024/2025  
**Fecha:** 26 de Noviembre de 2025
