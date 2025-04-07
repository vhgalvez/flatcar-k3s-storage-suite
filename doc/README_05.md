# flatcar-k3s-storage-suite

## 📦 Aplicaciones que usan almacenamiento

Esta sección documenta detalladamente qué aplicaciones dentro del clúster Kubernetes utilizan almacenamiento persistente, qué tipo de PVC requieren, cuál es el sistema de almacenamiento asociado (NFS o Longhorn) y en qué ruta se almacena la información en el nodo `storage1`. Además, se incluyen recomendaciones para el uso de tokens JWT en balanceadores.

| Aplicación            | Tipo PVC  | Sistema    | Ruta de Almacenamiento                     | Descripción Técnica                                                                 |
|-----------------------|-----------|------------|---------------------------------------------|--------------------------------------------------------------------------------------|
| PostgreSQL (externo)  | RWX       | NFS        | /srv/nfs/postgresql                         | Base de datos externa que consume almacenamiento compartido por NFS                |
| Prometheus            | RWO       | Longhorn   | /mnt/longhorn-disk/prometheus/             | Monitoreo. Usa almacenamiento replicado local para series temporales               |
| Grafana               | RWO       | Longhorn   | /mnt/longhorn-disk/grafana/                | Visualización de métricas. Volumen aislado mediante PVC                            |
| Elasticsearch         | RWO       | Longhorn   | /mnt/longhorn-disk/elasticsearch/          | Indexación y logs. Necesita almacenamiento de alto rendimiento replicado           |
| Redis                 | RWO       | Longhorn   | /mnt/longhorn-disk/redis/                  | Base de datos en memoria. PVC dedicado                                             |
| Kafka                 | RWO       | Longhorn   | /mnt/longhorn-disk/kafka/                  | Cola de eventos distribuida. PVC con durabilidad local                             |
| Nginx static assets   | RWX       | NFS        | /srv/nfs/shared/static/                    | Archivos estáticos accesibles por múltiples pods simultáneamente                  |
| Token JWT Traefik     | RWO/RWX   | Longhorn/NFS| /mnt/longhorn-disk/tokens/traefik.jwt      | Token de autenticación para el Ingress Controller. Se prepara ruta anticipadamente |

> ℹ️ **Nota:** Aunque el token JWT de Traefik se genera más adelante en su proceso de instalación, **se debe anticipar la creación del directorio** de almacenamiento para que esté disponible cuando se cree el archivo.

---

## 🧠 Almacenamiento Total Aproximado

Resumen de la capacidad y acceso ofrecido por cada sistema de almacenamiento:

| Sistema      | Capacidad Total | Tipo de Acceso  | Uso Principal                             |
|--------------|------------------|------------------|--------------------------------------------|
| Longhorn     | 120 GB (3x40 GB) | ReadWriteOnce    | Aplicaciones críticas con volumen único y replicación local |
| NFS          | 80 GB (LVM)      | ReadWriteMany    | Archivos compartidos, PostgreSQL, backups, assets públicos  |

---

## 📂 Rutas de Almacenamiento en `storage1`

El nodo `storage1` actúa como servidor NFS y punto de respaldo para volúmenes Longhorn. Las rutas montadas se preparan automáticamente con permisos `0777` por Ansible.

| Ruta                       | Tamaño Aproximado | Tipo      | Uso                                         |
|----------------------------|-------------------|-----------|----------------------------------------------|
| /srv/nfs/postgresql        | 10 GB             | NFS (RWX) | Base de datos PostgreSQL                    |
| /srv/nfs/shared            | 9 GB              | NFS (RWX) | Volúmenes compartidos entre pods            |
| /mnt/longhorn-disk         | 58 GB             | LVM       | Volúmenes RWO (Prometheus, Redis, etc.)     |
| /mnt/longhorn-disk/tokens/ | -                 | LVM       | Token JWT de Traefik (a generar después)    |

---

## 🔀 Persistencia y Alta Disponibilidad

| Escenario                         | Comportamiento Esperado                                              |
|----------------------------------|------------------------------------------------------------------------|
| Pod reiniciado                   | El PVC se vuelve a montar automáticamente en el mismo nodo             |
| Nodo worker caído                | Longhorn reatacha el volumen en otro nodo disponible                   |
| Reinicio de clúster              | Los volúmenes permanecen disponibles si la configuración fue correcta  |
| NFS Down                         | Los volúmenes RWX quedan inaccesibles temporalmente                    |

> 🛡️ Se recomienda monitorizar el estado de Longhorn y NFS con Prometheus y alertas. Longhorn provee UI de estado.

---

## 🖥️ Detalle de nodos y almacenamiento

| Nodo           | Rol                   | IP            | Disco OS | Disco Extra | Uso del Disco Extra                                         |
|----------------|------------------------|----------------|----------|-------------|-------------------------------------------------------------|
| master1        | Master Kubernetes      | 10.17.4.21     | 50 GB    | —           | —                                                           |
| master2        | Master Kubernetes      | 10.17.4.22     | 50 GB    | —           | —                                                           |
| master3        | Master Kubernetes      | 10.17.4.23     | 50 GB    | —           | —                                                           |
| worker1        | Worker + Longhorn      | 10.17.4.24     | 20 GB    | 40 GB       | Almacenamiento Longhorn (RWO)                               |
| worker2        | Worker + Longhorn      | 10.17.4.25     | 20 GB    | 40 GB       | Almacenamiento Longhorn (RWO)                               |
| worker3        | Worker + Longhorn      | 10.17.4.26     | 20 GB    | 40 GB       | Almacenamiento Longhorn (RWO)                               |
| storage1       | NFS + Longhorn Backup  | 10.17.4.27     | 10 GB    | 80 GB       | PostgreSQL, carpetas compartidas, tokens y backups          |

> 📎 Todos los discos adicionales están montados como `/dev/vdb` y gestionados vía Ansible con LVM. El montaje es automático y persistente.

---

## 🔐 Recomendaciones para el Token JWT de Traefik

El archivo del token JWT se debe generar durante la instalación de Traefik:

```bash
kubectl create token traefik-sa
```

### 📁 Preparar su almacenamiento anticipadamente

Antes de instalar Traefik, se deben crear los directorios de almacenamiento:

```bash
# Opción Longhorn
mkdir -p /mnt/longhorn-disk/tokens/
chmod 0777 /mnt/longhorn-disk/tokens/

# Opción NFS (compartido)
mkdir -p /srv/nfs/traefik-token
chmod 0777 /srv/nfs/traefik-token
```

Esto facilita que balanceadores múltiples puedan acceder al token JWT, ya sea vía Longhorn o NFS RWX.

---

## 📜 Licencia
Este proyecto está bajo la Licencia MIT. Para más detalles, consulta el archivo LICENSE.
