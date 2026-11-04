# Fase 2 — Configuración y validación de Podman en RHEL 9.8

## 1. Objetivo

Preparar la máquina virtual AWS para ejecutar los servicios de la PoC de IBM API Connect mediante contenedores Podman.

Los objetivos específicos de esta fase fueron:

- Verificar la instalación y el funcionamiento de Podman.
- Evaluar la posibilidad de ejecutar contenedores sin privilegios (*rootless*).
- Identificar y resolver las restricciones del entorno.
- Configurar el almacenamiento de imágenes y contenedores sobre el disco secundario de 1000 GB.
- Validar la ejecución real de un contenedor.
- Dejar el entorno preparado para desplegar los servicios de la aplicación.

**Estado: COMPLETADO**

---

## 2. Contexto de infraestructura

La infraestructura fue preparada durante la Fase 1.

| Componente | Configuración |
|---|---|
| Proveedor | AWS |
| Región | us-west-1 (California) |
| Sistema operativo | Red Hat Enterprise Linux 9.8 |
| Usuario SSH | `itzuser` |
| Dirección IP pública | `54.151.67.159` |
| CPU | 8 vCPU |
| RAM | 32 GB aproximadamente |
| Disco principal | 20 GB |
| Disco secundario | 1000 GB |
| Montaje secundario | `/data` |
| Directorio de proyecto | `/data/retail-poc` |
| Versión de Podman | 5.8.2 |

### Arquitectura prevista

La aplicación se desplegará con los siguientes servicios:

| Servicio | Tecnología | Despliegue |
|---|---|---|
| Backend | Python + FastAPI | Contenedor Podman |
| Base de datos | PostgreSQL | Contenedor Podman |
| Identity Provider | Keycloak | Contenedor Podman |
| Reverse proxy | Nginx | Contenedor Podman |
| API Management | IBM API Connect v12 | SaaS Frankfurt |
| API Gateway | IBM DataPower API Gateway | SaaS Frankfurt |
| Analytics | API Connect Analytics | SaaS Frankfurt |

Los cuatro primeros servicios se ejecutarán en la VM AWS. Los componentes de IBM API Connect estarán alojados en la plataforma SaaS de IBM.

---

## 3. Verificación inicial de Podman

Durante la Fase 1 se confirmó que Podman ya estaba instalado.

```bash
podman --version
```

Resultado:

```text
podman version 5.8.2
```

Se intentó obtener su configuración completa:

```bash
podman info
```

Resultado:

```text
cannot clone: No space left on device
user namespaces are not enabled in /proc/sys/user/max_user_namespaces
Error: cannot re-exec process
```

### Diagnóstico inicial

El mensaje sugería que el sistema tenía deshabilitada la creación de *user namespaces*, necesarios para ejecutar Podman en modo rootless.

El texto `No space left on device` no demostraba que el almacenamiento estuviera lleno. En este caso, el error estaba asociado a la imposibilidad de crear un nuevo namespace.

---

## 4. Diagnóstico de los user namespaces

Se consultó el límite de *user namespaces*:

```bash
cat /proc/sys/user/max_user_namespaces
```

Resultado:

```text
0
```

Posteriormente, se consultaron dos parámetros del kernel:

```bash
sysctl user.max_user_namespaces user.max_pid_namespaces
```

Resultado:

```text
user.max_user_namespaces = 0
user.max_pid_namespaces = 126956
```

### Interpretación

- `user.max_user_namespaces = 0`: impide crear nuevos *user namespaces*.
- `user.max_pid_namespaces = 126956`: existe capacidad configurada para crear *PID namespaces*.

El principal impedimento para ejecutar Podman rootless era el primer parámetro.

### Identificación de la configuración persistente

Se ejecutó:

```bash
sudo grep -RnsE '^\s*user\.max_user_namespaces\s*=' /etc/sysctl.conf /etc/sysctl.d /usr/lib/sysctl.d /run/sysctl.d 2>/dev/null
```

Resultado:

```text
/etc/sysctl.conf:41:user.max_user_namespaces=0
/etc/sysctl.d/99-sysctl.conf:41:user.max_user_namespaces=0
```

Se verificó si ambos archivos eran independientes:

```bash
ls -l /etc/sysctl.conf /etc/sysctl.d/99-sysctl.conf
```

Resultado relevante:

```text
/etc/sysctl.conf
/etc/sysctl.d/99-sysctl.conf -> ../sysctl.conf
```

Se confirmó que `/etc/sysctl.d/99-sysctl.conf` es un enlace simbólico al archivo `/etc/sysctl.conf`.

Por lo tanto, existe una única configuración efectiva que establece el límite en cero.

### Decisión técnica

Se decidió **no modificar `user.max_user_namespaces`**, dado que la VM pertenece a un entorno de workshop y la restricción podría haber sido establecida deliberadamente.

En su lugar, se evaluó ejecutar Podman con privilegios de administrador (*rootful*).

---

## 5. Validación de Podman rootful

Se ejecutó:

```bash
sudo podman info
```

El comando se completó correctamente.

### Información relevante

| Parámetro | Resultado |
|---|---|
| Podman | 5.8.2 |
| Arquitectura | amd64 |
| Sistema operativo | RHEL 9.8 |
| Runtime OCI | crun 1.27 |
| Cgroup Manager | systemd |
| Cgroup Version | v2 |
| Networking Backend | netavark |
| Database Backend | sqlite |
| Graph Driver | overlay |
| Rootless | false |
| SELinux habilitado según Podman | false |
| Contenedores existentes | 0 |
| Imágenes existentes | 0 |

El entorno dispone de 8 vCPU y aproximadamente 32 GB de RAM.

### Conclusión

Podman funciona correctamente utilizando `sudo`.

Se adoptó el modo rootful para la PoC, evitando modificaciones adicionales en los parámetros del kernel.

Esta decisión implica administrar cuidadosamente los privilegios de los contenedores, las redes y los puertos expuestos.

**Observación de seguridad:** `selinuxEnabled: false` es lo informado por Podman. No se verificó por separado el estado general de SELinux mediante `getenforce`.

---

## 6. Análisis del almacenamiento

El resultado de `sudo podman info` mostró inicialmente:

```text
graphRoot: /var/lib/containers/storage
graphRootAllocated: 20122152960
graphRootUsed: 9273278464
```

Esto indicaba que Podman utilizaba el disco principal, de aproximadamente 20 GB.

Durante la Fase 1 ya se había configurado un disco secundario de 1000 GB montado en:

```text
/data
```

Se decidió mover el almacenamiento persistente de Podman a este disco para evitar consumir innecesariamente el espacio disponible en el sistema operativo.

### Configuración original

Se ejecutó:

```bash
sudo grep -nE '^[[:space:]]*(\[storage\]|driver[[:space:]]*=|graphroot[[:space:]]*=|runroot[[:space:]]*=)' /etc/containers/storage.conf
```

Resultado:

```text
14:[storage]
17:driver = "overlay"
20:runroot = "/run/containers/storage"
32:graphroot = "/var/lib/containers/storage"
```

Significado de los parámetros:

| Parámetro | Función |
|---|---|
| `driver` | Tecnología utilizada para administrar las capas de almacenamiento |
| `runroot` | Información temporal de ejecución |
| `graphroot` | Almacenamiento persistente de imágenes, capas y contenedores |

Se decidió modificar únicamente `graphroot`.

---

## 7. Configuración del nuevo almacenamiento

### 7.1 Crear el directorio de almacenamiento

```bash
sudo mkdir -p /data/podman-storage
```

Se creó el directorio destinado a las imágenes y capas persistentes de los contenedores.

### 7.2 Respaldar la configuración original

Antes de modificar el archivo de configuración:

```bash
sudo cp -a /etc/containers/storage.conf /etc/containers/storage.conf.bak
```

Se generó la copia:

```text
/etc/containers/storage.conf.bak
```

### 7.3 Modificar graphroot

Se ejecutó:

```bash
sudo sed -i 's|^graphroot = "/var/lib/containers/storage"|graphroot = "/data/podman-storage"|' /etc/containers/storage.conf
```

Este comando reemplazó la ubicación anterior por la nueva.

Configuración deseada:

```ini
[storage]
driver = "overlay"
runroot = "/run/containers/storage"
graphroot = "/data/podman-storage"
```

No se modificaron el driver ni el directorio de ejecución temporal.

### 7.4 Verificar la configuración efectiva

Se ejecutó:

```bash
sudo podman info --format '{{.Store.GraphRoot}}'
```

Resultado:

```text
/data/podman-storage
```

**Validación exitosa.**

Podman reconoce el nuevo directorio de almacenamiento.

No fue necesario migrar contenedores ni imágenes, dado que el entorno se encontraba vacío antes del cambio.

---

## 8. Validación de ejecución de contenedores

Para comprobar que Podman podía descargar una imagen, crear un contenedor y ejecutarlo, se utilizó Red Hat Universal Base Image 9 Minimal.

Comando:

```bash
sudo podman run --rm registry.access.redhat.com/ubi9/ubi-minimal:latest /bin/sh -c 'echo "Podman funciona correctamente"'
```

### Resultado

```text
Trying to pull registry.access.redhat.com/ubi9/ubi-minimal:latest...
Getting image source signatures
Checking if image destination supports signatures
Copying blob 0811bc98b695 done
Copying config d2c7f8e178 done
Writing manifest to image destination
Storing signatures
Podman funciona correctamente
```

### Interpretación

La prueba confirmó que:

1. La VM puede acceder al registro de imágenes de Red Hat.
2. Podman puede descargar imágenes desde Internet.
3. El runtime OCI puede crear y ejecutar contenedores.
4. Los comandos dentro del contenedor se ejecutan correctamente.
5. El contenedor temporal finalizó sin errores.

La opción `--rm` eliminó automáticamente el contenedor después de ejecutarlo, pero conservó la imagen descargada.

La prueba se realizó con éxito utilizando Podman rootful.

---

## 9. Estado final de Podman

| Componente | Estado |
|---|---|
| Instalación de Podman | Completada |
| Versión 5.8.2 | Verificada |
| Podman rootless | Bloqueado por configuración del kernel |
| Podman rootful | Operativo |
| Almacenamiento en `/data` | Configurado |
| Driver overlay | Disponible |
| Runtime crun | Disponible |
| Network backend netavark | Disponible |
| Acceso al registro Red Hat | Verificado |
| Descarga de imágenes | Verificada |
| Ejecución de contenedores | Verificada |
| PostgreSQL | Pendiente |
| FastAPI | Pendiente |
| Keycloak | Pendiente |
| Nginx | Pendiente |
| Podman Compose | Pendiente de verificación |

### Configuración final relevante

```text
Sistema operativo: RHEL 9.8
Usuario SSH: itzuser

Podman:
  Version: 5.8.2
  Execution: rootful
  Runtime: crun
  Storage driver: overlay
  Graphroot: /data/podman-storage
  Runroot: /run/containers/storage
  Networking: netavark

Proyecto:
  /data/retail-poc

Almacenamiento:
  Disco principal: 20 GB
  Disco secundario: 1000 GB
  Punto de montaje: /data
```

---

## 10. Consideraciones técnicas

### Seguridad

Se decidió utilizar Podman rootful por las restricciones presentes en el kernel.

Esta modalidad exige prestar especial atención a:

- Evitar contenedores privilegiados salvo necesidad justificada.
- No exponer PostgreSQL ni los puertos administrativos de Keycloak públicamente.
- Administrar los secretos y credenciales fuera del repositorio Git.
- Configurar cuidadosamente los puertos publicados.
- Verificar el estado efectivo de SELinux y del firewall antes de exponer servicios.

No se deshabilitó SELinux ni se modificó su configuración durante esta fase.

### Persistencia

El almacenamiento de Podman se configuró en `/data/podman-storage`.

Los datos persistentes de la aplicación, especialmente PostgreSQL, se planificarán separadamente bajo `/data/retail-poc`.

También se deberá verificar que `/data` esté montado antes de iniciar los servicios, especialmente después de reiniciar la máquina.

### Gestión del entorno

Hasta este punto no se han instalado PostgreSQL, Keycloak, Nginx ni el backend FastAPI.

Tampoco se han definido todavía redes de contenedores, volúmenes de aplicación ni servicios persistentes.

---

## 11. Próximos pasos — Fase 3

El siguiente objetivo será desplegar los servicios de la aplicación de inventario.

**Paso inmediato pendiente: verificar disponibilidad de Podman Compose.**

Comando previsto, todavía no ejecutado:

```bash
sudo podman compose version
```

Posteriormente:

1. Definir la estructura de despliegue mediante Compose.
2. Seleccionar una imagen y versión de PostgreSQL.
3. Crear el almacenamiento persistente de la base de datos.
4. Configurar credenciales mediante mecanismos apropiados para secretos.
5. Desplegar PostgreSQL sin publicar su puerto a Internet.
6. Construir el backend FastAPI.
7. Integrar FastAPI con PostgreSQL.
8. Desplegar Keycloak para gestionar OAuth 2.0 y OpenID Connect.
9. Configurar Nginx como reverse proxy.
10. Preparar la publicación de las APIs mediante IBM API Connect SaaS.

## 12. Conclusión

Se completó correctamente la preparación de Podman sobre RHEL 9.8.

Se identificó una restricción del kernel que impedía utilizar contenedores rootless y se adoptó Podman rootful como alternativa para este entorno de workshop.

También se trasladó el almacenamiento persistente de Podman al disco secundario de 1000 GB y se validó exitosamente la descarga y ejecución de una imagen oficial de Red Hat.

**La infraestructura está preparada para comenzar el despliegue de los servicios contenedorizados de la PoC de IBM API Connect.**
