# Fase 1 — Preparación de infraestructura RHEL

## 1. Objetivo

Preparar una máquina virtual AWS con Red Hat Enterprise Linux para alojar los componentes de una PoC de IBM API Connect orientada al sector retail.

La infraestructura alojará:

- Backend de gestión de inventario.
- Base de datos PostgreSQL.
- Identity Provider (Keycloak).
- Reverse proxy (Nginx).
- Servicios auxiliares.

Los componentes se desplegarán mediante contenedores utilizando Podman.

## 2. Información de la VM

| Parámetro | Valor |
|---|---|
| Cloud provider | AWS |
| Región | us-west-1 (California) |
| Sistema operativo | Red Hat Enterprise Linux 9.8 |
| Arquitectura | x86_64 |
| Kernel | 5.14.0-687.51.1.el9_8.x86_64 |
| Usuario | itzuser |
| Hostname | ip-10-0-40-141.us-west-1.compute.internal |
| IP pública | 54.151.67.159 |
| IP privada | 10.0.40.141 |
| CPU | 8 vCPU |
| RAM | 32 GB |
| Disco principal | 20 GB |
| Disco secundario | 1000 GB |
| Container runtime | Podman 5.8.2 |

## 3. Verificación inicial del sistema

### 3.1. Identificar usuario

```bash
whoami
```

Resultado esperado:

```text
itzuser
```

### 3.2. Identificar sistema operativo y hostname

```bash
hostnamectl
```

### 3.3. Verificar versión de RHEL

```bash
cat /etc/redhat-release
```

Resultado:

```text
Red Hat Enterprise Linux release 9.8 (Plow)
```

### 3.4. Verificar kernel

```bash
uname -r
```

Resultado:

```text
5.14.0-687.51.1.el9_8.x86_64
```

## 4. Verificación de recursos

### 4.1. CPU

```bash
lscpu
```

Se verificaron 8 CPU virtuales Intel Xeon.

### 4.2. Memoria

```bash
free -h
```

Resultado aproximado:

```text
Mem: 31Gi total
Swap: 4Gi
```

### 4.3. Identificación de discos

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT,MODEL
```

Se identificaron:

- `/dev/xvda`: disco principal de 20 GB.
- `/dev/xvdh`: disco secundario de 1000 GB.

**Importante:** AWS presenta el disco como `/dev/xvdh`, aunque originalmente estuviera identificado como `/dev/sdh` en la configuración de infraestructura.

### 4.4. Verificar espacio disponible

```bash
df -hT
```

Se comprobó que el sistema operativo está instalado en `/dev/xvda4`, con filesystem XFS.

## 5. Preparación del disco secundario

### 5.1. Inspección inicial

Antes de modificar el disco, se verificó que estuviera vacío y sin montar.

```bash
sudo fdisk -l /dev/xvdh
```

```bash
sudo wipefs -n /dev/xvdh
```

```bash
findmnt
```

```bash
sudo blkid /dev/xvdh
```

No se encontraron particiones, firmas de filesystem ni puntos de montaje asociados.

### 5.2. Crear tabla de particiones GPT

```bash
sudo parted /dev/xvdh --script mklabel gpt
```

### 5.3. Crear partición principal

```bash
sudo parted /dev/xvdh --script mkpart primary xfs 0% 100%
```

### 5.4. Verificar partición

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT
```

Resultado esperado:

```text
xvdh     1000G disk
└─xvdh1  1000G part
```

### 5.5. Formatear con XFS

```bash
sudo mkfs.xfs -f /dev/xvdh1
```

Se creó correctamente un filesystem XFS sobre `/dev/xvdh1`.

**Advertencia:** el formateo es una operación destructiva. No debe repetirse sobre un disco que contenga datos que deban preservarse.

## 6. Montaje del disco

### 6.1. Crear directorio de montaje

```bash
sudo mkdir -p /data
```

### 6.2. Obtener UUID

```bash
sudo blkid /dev/xvdh1
```

UUID obtenido:

```text
7b173ece-0341-4757-bf2f-e0ffa1fc688d
```

### 6.3. Montar filesystem

```bash
sudo mount /dev/xvdh1 /data
```

### 6.4. Verificar montaje

```bash
df -hT /data
```

Resultado:

```text
Filesystem     Type  Size  Used Avail Use%
/dev/xvdh1     xfs  1000G  7.1G  993G   1%
```

## 7. Configuración de montaje persistente

Se configuró `/etc/fstab` para que el disco se monte automáticamente durante el arranque.

### 7.1. Crear copia de seguridad

```bash
sudo cp /etc/fstab /etc/fstab.backup
```

### 7.2. Editar configuración

```bash
sudo vi /etc/fstab
```

Se agregó la siguiente línea:

```text
UUID=7b173ece-0341-4757-bf2f-e0ffa1fc688d /data xfs defaults,nofail 0 0
```

### 7.3. Recargar configuración de systemd

```bash
sudo systemctl daemon-reload
```

### 7.4. Probar montaje automático

Se desmontó el filesystem:

```bash
sudo umount /data
```

Se ejecutó el montaje definido en `/etc/fstab`:

```bash
sudo mount -a
```

Se verificó:

```bash
findmnt /data
```

```bash
df -hT /data
```

El disco volvió a montarse correctamente en `/data`.

## 8. Crear estructura del proyecto

Se decidió centralizar los archivos de la PoC en el disco secundario para evitar consumir espacio innecesariamente en el disco principal.

### 8.1. Crear directorio principal

```bash
sudo mkdir -p /data/retail-poc
```

### 8.2. Asignar permisos al usuario

```bash
sudo chown -R itzuser:itzuser /data/retail-poc
```

### 8.3. Crear subdirectorios

```bash
mkdir -p /data/retail-poc/{backend,database,keycloak,nginx,apic,logs}
```

### 8.4. Verificar estructura

```bash
ls -la /data/retail-poc
```

Estructura resultante:

```text
/data/retail-poc/
├── backend/
├── database/
├── keycloak/
├── nginx/
├── apic/
└── logs/
```

### 8.5. Verificar almacenamiento

```bash
df -h /data
```

## 9. Verificación de Podman

Se comprobó que Podman ya estaba instalado en la VM.

### 9.1. Verificar versión

```bash
podman --version
```

Resultado:

```text
podman version 5.8.2
```

### 9.2. Verificar paquete RPM

```bash
rpm -q podman
```

Resultado:

```text
podman-5.8.2-7.el9_8.x86_64
```

### 9.3. Verificar repositorios disponibles

```bash
sudo dnf repolist
```

Se encontraron los siguientes repositorios:

- EPEL 9.
- EPEL Cisco OpenH264.
- RHEL 9 AppStream mediante RHUI.
- RHEL 9 BaseOS mediante RHUI.
- RHUI Client Configuration.

La VM no está registrada directamente en Red Hat Subscription Management, pero tiene acceso a los repositorios de AWS RHUI.

Esto debe tenerse en cuenta si posteriormente se necesitan instalar paquetes adicionales.

## 10. Estado final de la infraestructura

| Componente | Estado |
|---|---|
| RHEL 9.8 | Verificado |
| CPU y RAM | Verificados |
| Disco principal | Verificado |
| Disco secundario | Formateado XFS |
| Punto de montaje `/data` | Configurado |
| Montaje persistente | Configurado y probado |
| Directorio del proyecto | Creado |
| Permisos de usuario | Configurados |
| Podman | Instalado |
| Repositorios DNF | Verificados |

## 11. Pendientes para la siguiente fase

Todavía no se instalaron ni desplegaron los servicios de la aplicación.

Próximas actividades:

1. Verificar funcionamiento de Podman.
2. Evaluar instalación de herramientas complementarias.
3. Configurar almacenamiento persistente para contenedores.
4. Preparar PostgreSQL.
5. Desarrollar backend con FastAPI.
6. Desplegar Keycloak.
7. Configurar Nginx y HTTPS.
8. Publicar la API mediante IBM API Connect SaaS.

---

**Fin de Fase 1 — Preparación inicial de VM y almacenamiento.**
