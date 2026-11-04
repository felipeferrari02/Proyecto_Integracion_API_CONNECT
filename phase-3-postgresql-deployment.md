# Fase 3 — Despliegue de PostgreSQL con Podman Compose

**Proyecto:** Retail Inventory PoC — IBM API Connect SaaS  
**Fecha:** 04/11/2026  
**Sistema operativo:** Red Hat Enterprise Linux 9.8  
**Estado:** ✅ Completado

---

## 1. Objetivo

Desplegar PostgreSQL como primer servicio de nuestra aplicación de inventario, que posteriormente será integrada con IBM API Connect SaaS v12.

Durante esta fase se realizaron las siguientes tareas:

- Instalación de Podman Compose.
- Generación segura de credenciales para PostgreSQL.
- Configuración de secretos mediante Podman Secrets.
- Creación de un archivo `compose.yaml`.
- Despliegue de PostgreSQL 17 en un contenedor.
- Configuración de almacenamiento persistente.
- Validación de la base de datos, el usuario y la autenticación TCP.

## 2. Infraestructura disponible

La máquina virtual fue preparada durante las fases anteriores.

| Componente | Configuración |
|---|---|
| Proveedor | AWS |
| Región | us-west-1 |
| Sistema operativo | RHEL 9.8 |
| Usuario SSH | `itzuser` |
| CPU | 8 vCPU |
| RAM | 32 GB |
| Disco principal | 20 GB |
| Disco secundario | 1000 GB |
| Punto de montaje | `/data` |
| Podman | 5.8.2 |
| Almacenamiento Podman | `/data/podman-storage` |
| Directorio del proyecto | `/data/retail-poc` |

Podman se ejecuta mediante `sudo` (modo rootful), debido a que los *user namespaces* están deshabilitados en esta VM.

---

## 3. Instalación de Podman Compose

### 3.1 Verificación inicial

Se ejecutó:

```bash
sudo podman compose version
```

El sistema indicó que no encontraba un proveedor Compose:

```text
Error: looking up compose provider failed
```

Podman estaba correctamente instalado, pero no disponía de `podman-compose` ni de `docker-compose`.

### 3.2 Comprobar disponibilidad

```bash
sudo dnf info podman-compose
```

Resultado:

| Parámetro | Valor |
|---|---|
| Paquete | podman-compose |
| Versión | 1.5.0 |
| Repositorio | EPEL |
| Arquitectura | noarch |

### 3.3 Instalación

```bash
sudo dnf install podman-compose
```

Se instalaron los siguientes paquetes:

- `podman-compose-1.5.0`
- `python3-click`
- `python3-dotenv`
- `python3-dotenv+cli`

La instalación finalizó correctamente.

### 3.4 Validación

```bash
sudo podman compose version
```

Resultado:

```text
podman-compose version 1.5.0
podman version 5.8.2
```

**Conclusión:** Podman Compose quedó operativo y disponible para administrar los servicios de nuestra aplicación.

---

## 4. Configuración de credenciales

Para evitar contraseñas escritas directamente en los archivos de configuración, se utilizó el mecanismo nativo de secretos de Podman.

### 4.1 Crear el directorio de secretos

```bash
mkdir -m 700 -p /data/retail-poc/secrets
```

Verificación:

```bash
ls -ld /data/retail-poc/secrets
```

Resultado:

```text
drwx------ 2 itzuser itzuser
```

El directorio tiene permisos `700`, restringiendo el acceso a su propietario y a usuarios con privilegios administrativos suficientes.

### 4.2 Generar la contraseña

Se creó una contraseña aleatoria de 32 bytes:

```bash
(umask 077; set -C; openssl rand -hex 32 > /data/retail-poc/secrets/postgres_password)
```

Este procedimiento:

- Genera una contraseña criptográficamente aleatoria.
- La guarda directamente en un archivo.
- Evita sobrescribir accidentalmente un archivo existente.
- No muestra su contenido en la terminal.

Verificación:

```bash
ls -l /data/retail-poc/secrets/postgres_password
```

Resultado:

```text
-rw------- 1 itzuser itzuser 65 ... postgres_password
```

El archivo tiene permisos `600`, por lo que solamente el propietario puede leerlo y modificarlo, salvo usuarios con privilegios administrativos.

### 4.3 Crear un secreto de Podman

```bash
sudo podman secret create retail_postgres_password /data/retail-poc/secrets/postgres_password
```

El comando registró correctamente el secreto.

Para verificarlo:

```bash
sudo podman secret inspect retail_postgres_password
```

Se confirmó:

| Propiedad | Resultado |
|---|---|
| Nombre | `retail_postgres_password` |
| Driver | `file` |
| Almacenamiento | `/data/podman-storage/secrets/filedriver` |
| Estado | Creado correctamente |

Podman administra el secreto de forma independiente del archivo Compose.

**Importante:** las contraseñas reales no deben compartirse ni incluirse en GitHub. Para reproducir el entorno, se deben generar nuevas credenciales utilizando los comandos anteriores.

---

## 5. Creación del archivo compose.yaml

Se creó el archivo:

```text
/data/retail-poc/compose.yaml
```

Su contenido es el siguiente:

```yaml
name: retail-poc

services:
  postgres:
    image: docker.io/library/postgres:17
    restart: unless-stopped

    environment:
      POSTGRES_DB: retail_inventory
      POSTGRES_USER: retail_app
      POSTGRES_PASSWORD_FILE: /run/secrets/retail_postgres_password

    secrets:
      - retail_postgres_password

    volumes:
      - postgres_data:/var/lib/postgresql/data

    networks:
      - retail_internal

    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U retail_app -d retail_inventory"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:

networks:
  retail_internal:
    internal: true

secrets:
  retail_postgres_password:
    external: true
```

### Explicación de la configuración

| Elemento | Función |
|---|---|
| `postgres:17` | Imagen oficial de PostgreSQL 17 |
| `POSTGRES_DB` | Nombre de la base de datos |
| `POSTGRES_USER` | Usuario de PostgreSQL |
| `POSTGRES_PASSWORD_FILE` | Archivo desde el cual se obtiene la contraseña |
| `postgres_data` | Volumen persistente |
| `retail_internal` | Red interna |
| `healthcheck` | Verificación automática del estado de PostgreSQL |
| `external: true` | Utiliza el secreto creado previamente en Podman |

**Seguridad de red:** no se configuró ninguna sección `ports`. Por lo tanto, PostgreSQL no publica el puerto 5432 hacia el host.

### Validación del archivo

Se verificó su contenido:

```bash
nl -ba /data/retail-poc/compose.yaml
```

Luego se validó la configuración:

```bash
sudo podman compose -f /data/retail-poc/compose.yaml config
```

El comando interpretó correctamente el archivo YAML, incluidos el servicio, la red, el volumen y la referencia al secreto externo.

---

## 6. Despliegue de PostgreSQL

Para descargar la imagen y levantar el contenedor:

```bash
sudo podman compose -f /data/retail-poc/compose.yaml up -d
```

Podman realizó las siguientes operaciones:

1. Descargó la imagen oficial `postgres:17` desde Docker Hub.
2. Creó los recursos necesarios.
3. Configuró el volumen persistente.
4. Preparó la red interna.
5. Inició el contenedor PostgreSQL.

Nombre del contenedor creado:

```text
retail-poc_postgres_1
```

### Verificación del estado

```bash
sudo podman ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

Resultado:

```text
NAMES                  STATUS                   PORTS
retail-poc_postgres_1  Up 27 seconds (healthy)  5432/tcp
```

El estado `healthy` confirmó que PostgreSQL estaba aceptando conexiones de acuerdo con el healthcheck configurado.

El valor `5432/tcp` representa el puerto del contenedor, no una publicación hacia Internet.

---

## 7. Verificación de la base de datos

Se realizó una consulta SQL desde dentro del contenedor:

```bash
sudo podman exec retail-poc_postgres_1 psql -U retail_app -d retail_inventory -c 'SELECT current_database(), current_user;'
```

Resultado:

```text
 current_database | current_user
------------------+--------------
 retail_inventory | retail_app
(1 row)
```

Se verificó que:

- La base de datos `retail_inventory` existe.
- El usuario `retail_app` existe.
- PostgreSQL permite ejecutar consultas SQL.

Esta primera prueba utilizó una conexión local dentro del contenedor.

---

## 8. Verificación del almacenamiento persistente

Durante la Fase 2 se configuró Podman para almacenar sus datos en el disco secundario de 1000 GB.

Para verificar dónde se guarda la base de datos:

```bash
sudo podman inspect retail-poc_postgres_1 --format '{{range .Mounts}}{{println .Source "->" .Destination}}{{end}}'
```

Resultado:

```text
/data/podman-storage/volumes/retail-poc_postgres_data/_data -> /var/lib/postgresql/data
```

Esto confirma que PostgreSQL utiliza un volumen persistente ubicado en `/data`.

| Ubicación | Descripción |
|---|---|
| `/data/podman-storage/volumes/retail-poc_postgres_data/_data` | Datos persistentes en AWS |
| `/var/lib/postgresql/data` | Directorio de datos dentro del contenedor |

La eliminación o recreación del contenedor no elimina automáticamente los datos, siempre que se conserve el volumen.

**Nota:** todavía no se implementó un procedimiento de backup y restauración.

---

## 9. Verificación de autenticación TCP

Para confirmar que la contraseña configurada mediante Podman Secrets funciona correctamente, se realizó una conexión TCP dentro del contenedor.

Comando:

```bash
sudo podman exec retail-poc_postgres_1 sh -c 'PGPASSWORD="$(cat /run/secrets/retail_postgres_password)" psql -h 127.0.0.1 -U retail_app -d retail_inventory -c "SELECT current_database(), current_user;"'
```

Este comando:

1. Obtiene la contraseña desde el secreto montado por Podman.
2. La utiliza temporalmente para la conexión.
3. Se conecta a PostgreSQL mediante TCP.
4. Se autentica con el usuario `retail_app`.
5. Ejecuta una consulta SQL.

Resultado:

```text
 current_database | current_user
------------------+--------------
 retail_inventory | retail_app
(1 row)
```

**Resultado exitoso:** PostgreSQL acepta conexiones TCP locales utilizando la contraseña configurada.

La conectividad desde otro contenedor todavía no fue verificada. La probaremos durante la integración con FastAPI.

---

## 10. Estado final de la Fase 3

| Componente | Estado |
|---|---|
| Podman Compose 1.5.0 | ✅ Instalado |
| Directorio de secretos | ✅ Creado |
| Contraseña de PostgreSQL | ✅ Generada |
| Secreto de Podman | ✅ Registrado |
| Archivo Compose | ✅ Validado |
| PostgreSQL 17 | ✅ Desplegado |
| Contenedor PostgreSQL | ✅ Healthy |
| Base de datos `retail_inventory` | ✅ Disponible |
| Usuario `retail_app` | ✅ Disponible |
| Volumen persistente | ✅ Verificado |
| Autenticación TCP | ✅ Validada |
| Integración con FastAPI | Pendiente |
| Backup de PostgreSQL | Pendiente |

### Estructura actual del proyecto

```text
/data/
├── podman-storage/
│   ├── volumes/
│   │   └── retail-poc_postgres_data/
│   └── secrets/
│
└── retail-poc/
    ├── compose.yaml
    ├── backend/
    ├── database/
    ├── keycloak/
    ├── nginx/
    ├── apic/
    ├── logs/
    └── secrets/
        └── postgres_password
```

La estructura representa los directorios y recursos relevantes de la PoC, no un listado exhaustivo del almacenamiento interno de Podman.

### Configuración PostgreSQL

| Parámetro | Valor |
|---|---|
| Versión | PostgreSQL 17 |
| Contenedor | `retail-poc_postgres_1` |
| Base de datos | `retail_inventory` |
| Usuario | `retail_app` |
| Puerto interno | 5432 |
| Contraseña | Podman Secret |
| Volumen | `retail-poc_postgres_data` |
| Red | `retail_internal` |

---

## 11. Consideraciones para reproducir el entorno

Esta documentación se guarda manualmente en GitHub. No existe ningún repositorio Git conectado a la VM.

Para reproducir esta fase en otra máquina, es necesario:

1. Tener Podman y Podman Compose instalados.
2. Configurar el almacenamiento persistente.
3. Crear el directorio de secretos.
4. Generar una nueva contraseña para PostgreSQL.
5. Registrar el secreto `retail_postgres_password`.
6. Crear el archivo `compose.yaml`.
7. Validar su configuración.
8. Ejecutar `podman compose up -d`.
9. Verificar el estado y la conectividad de PostgreSQL.

**Nunca se deben copiar al repositorio las contraseñas reales, claves privadas, tokens ni archivos del directorio `secrets/`.**

Si en el futuro el proyecto se administra mediante Git, se deberá utilizar un archivo `.gitignore` para excluir los secretos y otros archivos sensibles.

También quedan pendientes la validación de reinicio automático tras un reboot de la VM y la implementación de backups.

---

## 12. Próxima fase — Desarrollo de FastAPI

Con PostgreSQL funcionando, el siguiente objetivo será desarrollar nuestro backend REST de inventario utilizando Python y FastAPI.

### Funcionalidades previstas

| Método | Endpoint | Descripción |
|---|---|---|
| GET | `/health` | Estado de la aplicación |
| GET | `/products` | Listar productos |
| POST | `/products` | Crear producto |
| GET | `/products/{id}` | Consultar producto |
| PUT | `/products/{id}` | Actualizar producto |
| DELETE | `/products/{id}` | Eliminar producto |
| GET | `/inventory/{product_id}` | Consultar stock |
| POST | `/inventory/movements` | Registrar movimientos |

Los endpoints todavía no están implementados.

### Integración prevista

```text
FastAPI
   |
   | Conexión PostgreSQL / TCP 5432
   v
PostgreSQL
   |
   v
Base de datos retail_inventory
```

Posteriormente, expondremos los endpoints mediante Nginx y los publicaremos en IBM API Connect SaaS para demostrar seguridad, gestión de APIs, control de tráfico y Analytics.

---

## Conclusión

**La Fase 3 se completó correctamente el 4 de noviembre de 2026.**

Se instaló Podman Compose, se configuraron credenciales mediante Podman Secrets y se desplegó PostgreSQL 17 con almacenamiento persistente en el disco secundario.

Se comprobó que PostgreSQL está operativo, que la base de datos y el usuario existen y que la autenticación TCP mediante contraseña funciona correctamente.

**Próximo paso: Fase 4 — Desarrollo del backend Python + FastAPI.**
