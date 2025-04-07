## 📦 Aplicaciones que usan almacenamiento

Esta sección describe detalladamente qué aplicaciones requieren almacenamiento persistente, el tipo de volumen que usan (PVC), el sistema de almacenamiento utilizado (NFS o Longhorn), y la ruta física donde se almacenan sus datos. Esta documentación es esencial para anticipar las necesidades de espacio, replicación y acceso.

| Aplicación            | Tipo PVC  | Sistema     | Ruta de Almacenamiento                   | Descripción técnica                                                                 |
|-----------------------|-----------|-------------|-------------------------------------------|--------------------------------------------------------------------------------------|
| PostgreSQL (externo)  | RWX       | NFS         | /srv/nfs/postgresql                       | Base de datos montada como volumen compartido desde storage1                        |
| Prometheus            | RWO       | Longhorn    | /mnt/longhorn-disk/prometheus/           | Base de datos de métricas (TSDB)                                                    |
| Grafana               | RWO       | Longhorn    | /mnt/longhorn-disk/grafana/              | Dashboards y configuraciones                                                        |
| Elasticsearch         | RWO       | Longhorn    | /mnt/longhorn-disk/elasticsearch/        | Índices y datos persistentes                                                        |
| Redis                 | RWO       | Longhorn    | /mnt/longhorn-disk/redis/                | Almacenamiento en memoria persistente                                               |
| Kafka                 | RWO       | Longhorn    | /mnt/longhorn-disk/kafka/                | Logs y cola de mensajes distribuidos                                                |
| Nginx static assets   | RWX       | NFS         | /srv/nfs/shared/static/                  | HTML, imágenes y contenido estático compartido entre pods                           |
| Token JWT Traefik     | RWO/RWX   | Longhorn/NFS| /mnt/longhorn-disk/tokens/traefik.jwt    | Token de autenticación para el Ingress Controller externo (generado luego)         |

> ℹ️ **Nota:** Aunque el token JWT de Traefik se genera durante la instalación del Ingress Controller, es **altamente recomendable** dejar preparado el directorio correspondiente desde el inicio para facilitar el montaje posterior.

---

## 🧠 Almacenamiento Total Aproximado

Resumen de capacidad estimada por tipo de almacenamiento, incluyendo su acceso y propósito:

| Sistema      | Capacidad Total | Tipo de Acceso  | Uso Principal                                                       |
|--------------|------------------|------------------|----------------------------------------------------------------------|
| Longhorn     | 120 GB (3x40 GB) | ReadWriteOnce    | Aplicaciones críticas con necesidad de replicación o aislamiento    |
| NFS          | 80 GB (LVM)      | ReadWriteMany    | Aplicaciones que requieren acceso compartido o sincronizado         |

---

## 📂 Rutas de Almacenamiento en `storage1`

Este nodo sirve como **servidor principal de almacenamiento**. Las rutas aquí descritas son montadas con permisos `0777` y exportadas por Ansible mediante `/etc/exports`.

| Ruta                       | Tamaño Aproximado | Tipo      | Uso                                                                 |
|----------------------------|-------------------|-----------|----------------------------------------------------------------------|
| /srv/nfs/postgresql        | 10 GB             | NFS (RWX) | Base de datos accesible por varios pods                             |
| /srv/nfs/shared            | 9 GB              | NFS (RWX) | Compartir datos entre microservicios (cargas estáticas, configuraciones) |
| /mnt/longhorn-disk         | 58 GB             | LVM       | Volúmenes Longhorn: Prometheus, Redis, Kafka, etc.                  |
| /mnt/longhorn-disk/tokens/ | -                 | LVM       | Token JWT Traefik a generar durante instalación del Ingress         |

---

## 🔀 Persistencia y Alta Disponibilidad

El sistema está diseñado para ofrecer tolerancia a fallos y recuperación automática.

| Escenario                         | Comportamiento Esperado                                               |
|----------------------------------|------------------------------------------------------------------------|
| Pod reiniciado                   | El volumen Longhorn o NFS se vuelve a montar automáticamente          |
| Nodo worker caído                | Longhorn reatacha PVC a otro nodo disponible automáticamente          |
| Reinicio de clúster              | Los PVCs persisten si las rutas están correctamente definidas         |
| NFS Down                         | Las aplicaciones con PVC RWX quedan inaccesibles hasta restablecer NFS|

---

## 🖥️ Detalle de nodos y almacenamiento

| Nodo           | Rol                   | IP            | Disco OS | Disco Extra | Uso del Disco Extra                                                  |
|----------------|------------------------|----------------|----------|-------------|-------------------------------------------------------------------------|
| master1        | Master Kubernetes      | 10.17.4.21     | 50 GB    | —           | —                                                                       |
| master2        | Master Kubernetes      | 10.17.4.22     | 50 GB    | —           | —                                                                       |
| master3        | Master Kubernetes      | 10.17.4.23     | 50 GB    | —           | —                                                                       |
| worker1        | Worker + Longhorn      | 10.17.4.24     | 20 GB    | 40 GB       | Disco adicional usado por Longhorn como volumen local persistente      |
| worker2        | Worker + Longhorn      | 10.17.4.25     | 20 GB    | 40 GB       | Almacenamiento distribuido Longhorn (replicación automática)           |
| worker3        | Worker + Longhorn      | 10.17.4.26     | 20 GB    | 40 GB       | Igual que los anteriores, todos forman parte del clúster Longhorn      |
| storage1       | NFS + Longhorn Backup  | 10.17.4.27     | 10 GB    | 80 GB       | Exportaciones NFS, almacenamiento compartido, directorio tokens JWT    |

> 📌 Todos los discos adicionales están conectados como `/dev/vdb` y aprovisionados con Ansible vía LVM. Las configuraciones son reproducibles.

---

## 🔐 Preparación anticipada para el Token JWT de Traefik

Aunque el JWT de Traefik se genera en el paso de instalación del Ingress Controller, es una buena práctica dejar su directorio listo de antemano:

### 📁 Longhorn (RWO – persistente por pod)
```bash
mkdir -p /mnt/longhorn-disk/tokens/
chmod 0777 /mnt/longhorn-disk/tokens/
```

### 📁 Alternativa compartida con NFS (RWX – compartido entre pods o nodos)
```bash
mkdir -p /srv/nfs/traefik-token
chmod 0777 /srv/nfs/traefik-token
```

Esto permite que durante la instalación del Ingress Controller puedas montar directamente el JWT como volumen preexistente.

> ⚠️ **Importante:** No intentes crear el token JWT manualmente aún. Solo prepara el directorio. La generación se realiza con:
```bash
kubectl create token traefik-sa
```

---

✅ Con este almacenamiento preconfigurado, puedes continuar con el despliegue de tus herramientas de monitoreo, mensajería y bases de datos con volúmenes persistentes garantizados.
