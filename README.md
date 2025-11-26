# Práctica 2: Despliegue de aplicaciones usando Kubernetes/Docker
## Sistemas Distribuidos - Curso 2024/2025
## Alejandro Mamán López-Mingo
---

## 📋 Índice

1. [Descripción del Proyecto](#descripción-del-proyecto)
2. [Arquitectura Implementada](#arquitectura-implementada)
3. [Configuración Básica (COMPLETADA)](#configuración-básica-completada)
4. [Pasos para Llegar al 10](#pasos-para-llegar-al-10)
5. [Comandos Útiles](#comandos-útiles)
6. [Troubleshooting](#troubleshooting)

---

## 📖 Descripción del Proyecto

Sistema distribuido de gestión de archivos desplegado en un cluster Kubernetes con:
- **Broker:** Coordina las conexiones entre clientes y servidores
- **Servidores:** Gestionan archivos remotos (listado, subida, descarga)
- **Cliente:** Interfaz para interactuar con el sistema

### Componentes proporcionados:
- `brokerFileManager` - Ejecutable del broker
- `serverFileManager` - Ejecutable del servidor
- `clientFileManager` - Ejecutable del cliente

---

## 🏗️ Arquitectura Implementada

### Cluster Kubernetes

```
┌─────────────────────────────────────────────────────────────┐
│  NODO: k8smaster0.psdi.org (control-plane)                  │
│  IP: 172.31.64.84                                           │
│  Rol: Master + Puede ejecutar pods                          │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  NODO: k8sslave1.psdi.org (worker)                          │
│  IP: 172.31.31.30                                           │
│  Label: node-role=broker                                     │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  POD: broker-deployment                               │  │
│  │  Imagen: d1n0s/kubernetes-practica2broker:v1         │  │
│  │  Puerto: 32002                                        │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  NODO: k8sslave2.psdi.org (worker)                          │
│  IP: 172.31.72.209                                          │
│  Label: node-role=server                                     │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  POD: server-deployment                               │  │
│  │  Imagen: d1n0s/kubernetes-practica2server:v2         │  │
│  │  Puerto: 32001                                        │  │
│  │  Directorio: FileManagerDir/                          │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### Servicios Kubernetes

| Servicio | Tipo | ClusterIP | NodePort | Selector |
|----------|------|-----------|----------|----------|
| brokerservice | NodePort | 10.96.11.73 | 32002 | app=brokerfilemanager |
| serverservice | NodePort | 10.106.134.34 | 32001 | app=server-deploy |

---

## ✅ Configuración Básica (COMPLETADA)

### 1. Imágenes Docker Creadas

#### Imagen del Broker
```dockerfile
FROM ubuntu:20.04
RUN apt-get update
RUN apt-get install -y software-properties-common curl
EXPOSE 32002
COPY brokerFileManager /
RUN chmod +x /brokerFileManager
CMD /brokerFileManager
```

**Imagen en Docker Hub:** `d1n0s/kubernetes-practica2broker:v1`

#### Imagen del Servidor
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

**Imagen en Docker Hub:** `d1n0s/kubernetes-practica2server:v2`

**Nota:** El servidor se conecta al broker usando la IP `172.31.31.30` (k8sslave1)

### 2. Cluster Kubernetes Configurado

```bash
# Estado del cluster
kubectl get nodes -o wide
```

**Resultado:**
```
NAME                  STATUS   ROLES           AGE   VERSION    INTERNAL-IP
k8smaster0.psdi.org   Ready    control-plane   15d   v1.28.15   172.31.64.84
k8sslave1.psdi.org    Ready    worker          49m   v1.28.15   172.31.31.30
k8sslave2.psdi.org    Ready    worker          2m    v1.28.15   172.31.72.209
```

### 3. Deployments y Services Aplicados

#### Broker Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
 name: broker-deployment
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
    node-role: broker
   containers:
   - name: broker-deployment
     image: docker.io/d1n0s/kubernetes-practica2broker:v1
     ports:
     - containerPort: 32002
```

#### Server Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
 name: server-deployment
spec:
 replicas: 1
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
```

### 4. Verificación del Sistema

```bash
# Ver pods
kubectl get pods -o wide
```

**Resultado:**
```
NAME                                 READY   STATUS    NODE
broker-deployment-6fd556654c-jzdsx   1/1     Running   k8sslave1.psdi.org
server-deployment-689b756d6-6jqz8    1/1     Running   k8sslave2.psdi.org
```

### 5. Prueba del Cliente

```bash
# Conectar al broker (IP privada dentro del cluster)
./clientFileManager 172.31.31.30 32002
```

**Comandos disponibles:**
- `ls` - Lista archivos locales al cliente
- `lls` - Lista archivos en FileManagerDir/ del servidor
- `upload archivo.txt` - Sube archivo al servidor
- `download archivo.txt` - Descarga archivo del servidor
- `exit` - Cierra la conexión

**✅ CONFIGURACIÓN BÁSICA APROBADA**

---

## 🎯 Pasos para Llegar al 10

Para obtener la máxima calificación, debes implementar una de las dos configuraciones avanzadas:

### 📌 Opción 1: Múltiples Réplicas con Volumen Compartido (hostPath)

Esta configuración permite tener **múltiples réplicas del servidor en un mismo nodo** compartiendo la misma carpeta de archivos.

#### Paso 1: Crear PersistentVolume con hostPath

Crea el archivo `pv-hostpath.yml`:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: server-pv-hostpath
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  hostPath:
    path: /mnt/filemanager-data
    type: DirectoryOrCreate
  storageClassName: manual
```

#### Paso 2: Crear PersistentVolumeClaim

Crea el archivo `pvc-hostpath.yml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: server-pvc-hostpath
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: manual
```

#### Paso 3: Modificar DeploymentServer.yml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
 name: server-deployment
spec:
 replicas: 3  # ← Cambiar a 3 réplicas
 selector:
  matchLabels:
   app: server-deploy
 template:
  metadata:
   labels:
    app: server-deploy
  spec:
   nodeSelector:
    node-role: server  # Todas las réplicas en k8sslave2
   containers:
   - name: server-deployment
     image: docker.io/d1n0s/kubernetes-practica2server:v2
     ports:
     - containerPort: 32001
     volumeMounts:  # ← Añadir esto
     - name: filemanager-storage
       mountPath: /FileManagerDir
   volumes:  # ← Añadir esto
   - name: filemanager-storage
     persistentVolumeClaim:
       claimName: server-pvc-hostpath
```

#### Paso 4: Aplicar los cambios

```bash
# Aplicar PV y PVC
kubectl apply -f ~/Practica2/SERVER/pv-hostpath.yml
kubectl apply -f ~/Practica2/SERVER/pvc-hostpath.yml

# Verificar
kubectl get pv
kubectl get pvc

# Redesplegar servidor
kubectl delete deployment server-deployment
kubectl apply -f ~/Practica2/SERVER/DeploymentServer.yml

# Verificar que hay 3 réplicas
kubectl get pods -o wide
```

#### Paso 5: Probar persistencia

```bash
# Conectar al cliente
./clientFileManager 172.31.31.30 32002

# Subir un archivo
upload test.txt

# Listar archivos en el servidor
lls

# Salir y volver a conectar
exit
./clientFileManager 172.31.31.30 32002

# El archivo debe seguir ahí
lls
```

**Ventaja:** Las 3 réplicas comparten los mismos archivos. Si subes un archivo conectándote a una réplica, las otras también lo verán.

---

### 📌 Opción 2: Múltiples Nodos con NFS Compartido

Esta configuración permite tener **servidores distribuidos en múltiples nodos** compartiendo archivos mediante NFS.

#### Requisitos previos
- Añadir un tercer nodo worker (k8sslave3) al cluster
- Etiquetar k8sslave2 y k8sslave3 como `node-role=server`

#### Paso 1: Instalar servidor NFS en k8smaster0

```bash
# En k8smaster0
sudo apt-get update
sudo apt-get install -y nfs-kernel-server

# Crear directorio compartido
sudo mkdir -p /mnt/nfs-filemanager
sudo chown nobody:nogroup /mnt/nfs-filemanager
sudo chmod 777 /mnt/nfs-filemanager

# Configurar exports
echo "/mnt/nfs-filemanager *(rw,sync,no_subtree_check,no_root_squash)" | sudo tee -a /etc/exports

# Reiniciar NFS
sudo exportfs -ra
sudo systemctl restart nfs-kernel-server
```

#### Paso 2: Instalar cliente NFS en workers

```bash
# En k8sslave2 y k8sslave3
sudo apt-get update
sudo apt-get install -y nfs-common
```

#### Paso 3: Crear PersistentVolume NFS

Crea el archivo `pv-nfs.yml`:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: server-pv-nfs
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteMany
  nfs:
    server: 172.31.64.84  # IP del k8smaster0
    path: /mnt/nfs-filemanager
  storageClassName: nfs
```

#### Paso 4: Crear PersistentVolumeClaim NFS

Crea el archivo `pvc-nfs.yml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: server-pvc-nfs
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
  storageClassName: nfs
```

#### Paso 5: Modificar DeploymentServer.yml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
 name: server-deployment
spec:
 replicas: 3  # ← 3 réplicas distribuidas
 selector:
  matchLabels:
   app: server-deploy
 template:
  metadata:
   labels:
    app: server-deploy
  spec:
   nodeSelector:
    node-role: server  # Se distribuyen entre k8sslave2 y k8sslave3
   containers:
   - name: server-deployment
     image: docker.io/d1n0s/kubernetes-practica2server:v2
     ports:
     - containerPort: 32001
     volumeMounts:
     - name: filemanager-storage-nfs
       mountPath: /FileManagerDir
   volumes:
   - name: filemanager-storage-nfs
     persistentVolumeClaim:
       claimName: server-pvc-nfs
```

#### Paso 6: Aplicar configuración

```bash
# Aplicar PV y PVC NFS
kubectl apply -f ~/Practica2/SERVER/pv-nfs.yml
kubectl apply -f ~/Practica2/SERVER/pvc-nfs.yml

# Verificar
kubectl get pv
kubectl get pvc

# Redesplegar
kubectl delete deployment server-deployment
kubectl apply -f ~/Practica2/SERVER/DeploymentServer.yml

# Ver distribución de pods
kubectl get pods -o wide
```

**Ventaja:** Los servidores están en diferentes nodos físicos, pero todos comparten los mismos archivos mediante NFS.

---

## 🔧 Comandos Útiles

### Gestión del Cluster

```bash
# Ver estado de nodos
kubectl get nodes -o wide

# Ver pods
kubectl get pods -o wide
kubectl get pods -n kube-system

# Ver servicios
kubectl get svc

# Ver eventos
kubectl get events --sort-by='.lastTimestamp'
```

### Gestión de Deployments

```bash
# Ver deployments
kubectl get deployments

# Describir deployment
kubectl describe deployment broker-deployment
kubectl describe deployment server-deployment

# Ver logs de un pod
kubectl logs <nombre-pod>

# Eliminar deployment
kubectl delete deployment <nombre>

# Aplicar cambios
kubectl apply -f <archivo.yml>

# Escalar réplicas
kubectl scale deployment server-deployment --replicas=3
```

### Gestión de Imágenes Docker

```bash
# Construir imagen
docker build -t d1n0s/kubernetes-practica2broker:v1 -f DockerfileBroker .
docker build -t d1n0s/kubernetes-practica2server:v2 -f DockerfileServer .

# Subir a Docker Hub
docker login
docker push d1n0s/kubernetes-practica2broker:v1
docker push d1n0s/kubernetes-practica2server:v2

# Ver imágenes locales
docker images

# Eliminar imagen
docker rmi <imagen>
```

### Gestión de Nodos

```bash
# Etiquetar nodos
kubectl label nodes k8sslave1.psdi.org node-role=broker
kubectl label nodes k8sslave2.psdi.org node-role=server

# Ver etiquetas
kubectl get nodes --show-labels

# Añadir nodo worker
cd ~/kub
./kub_addNode.sh <IP>

# Eliminar nodo
kubectl drain <nombre-nodo> --ignore-daemonsets --delete-emptydir-data
kubectl delete node <nombre-nodo>
```

---

## 🐛 Troubleshooting

### Problema: Pods en CrashLoopBackOff

**Causa:** Error en la imagen Docker o configuración incorrecta.

**Solución:**
```bash
# Ver logs del pod
kubectl logs <nombre-pod>

# Describir el pod para ver eventos
kubectl describe pod <nombre-pod>
```

### Problema: No se puede conectar al broker desde fuera del cluster

**Causa:** Security group de AWS bloqueando el puerto 32002.

**Solución:**
1. Ve a AWS EC2 Console → Security Groups
2. Edita el security group de las instancias
3. Añade regla de entrada:
   - Tipo: Custom TCP
   - Puerto: 32002
   - Origen: 0.0.0.0/0

### Problema: Kubernetes no descarga la nueva imagen

**Causa:** Usa la imagen en caché con el mismo tag.

**Solución:**
```bash
# Cambiar el tag de la imagen
docker build -t d1n0s/kubernetes-practica2server:v3 .
docker push d1n0s/kubernetes-practica2server:v3

# Actualizar el deployment con el nuevo tag
# Editar DeploymentServer.yml: image: ...server:v3
kubectl apply -f DeploymentServer.yml
```

### Problema: Pods no se distribuyen en los nodos deseados

**Causa:** Falta `nodeSelector` o las etiquetas no coinciden.

**Solución:**
```bash
# Verificar etiquetas de los nodos
kubectl get nodes --show-labels

# Etiquetar correctamente
kubectl label nodes <nombre-nodo> node-role=<valor> --overwrite

# Añadir nodeSelector en el deployment
```

### Problema: PVC queda en estado Pending

**Causa:** No hay PV disponible o no coinciden las especificaciones.

**Solución:**
```bash
# Ver estado de PV y PVC
kubectl get pv
kubectl get pvc

# Describir para ver el error
kubectl describe pvc <nombre-pvc>

# Verificar que accessModes y storageClassName coinciden
```

---

## 📚 Referencias

- [Documentación oficial de Kubernetes](https://kubernetes.io/docs/home/)
- [Docker Hub](https://hub.docker.com/)
- [Guía de volúmenes NFS en Kubernetes](https://www.jorgedelacruz.es/2017/12/26/kubernetes-volumenes-nfs/)
- [Calico CNI](https://docs.projectcalico.org/)

---

## ✅ Checklist Final

### Configuración Básica
- [x] Cluster con 3 nodos (master + 2 workers)
- [x] Imágenes Docker creadas y subidas
- [x] Broker desplegado en k8sslave1
- [x] Servidor desplegado en k8sslave2
- [x] Servicios NodePort configurados
- [x] Cliente puede conectarse y ejecutar comandos

### Configuración Avanzada (Elige una)
- [ ] Config 1: Múltiples réplicas con hostPath
- [ ] Config 2: Múltiples nodos con NFS

---

## 👨‍💻 Autor

Alumno: [Tu Nombre]
Asignatura: Sistemas Distribuidos
Curso: 2024/2025
