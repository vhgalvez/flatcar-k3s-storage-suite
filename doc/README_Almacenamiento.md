# 📦 Arquitectura de Almacenamiento – flatcar-k3s-storage-suite

Este documento detalla cómo se gestiona TODO el almacenamiento persistente, distribuido, compartido y crítico dentro del entorno Kubernetes **FlatcarMicroCloud**, una infraestructura HA con nodos virtualizados, usando **Ansible, NFS, Longhorn, LVM, K3s** y múltiples servicios desplegados sobre **Flatcar Container Linux**.

---

## 💍 Nodos y Almacenamiento Asignado

| Nodo         | IP           | Rol                          | Disco OS | Disco adicional     | Uso del disco adicional                     |
|--------------|--------------|------------------------------|----------|----------------------|---------------------------------------------|
| master1      | 10.17.4.21   | K3s master (etcd)            | 50 GB    | —                    | —                                           |
| master2      | 10.17.4.22   | K3s master (etcd)            | 50 GB    | —                    | —                                           |
| master3      | 10.17.4.23   | K3s master (etcd)            | 50 GB    | —                    | —                                           |
| worker1      | 10.17.4.24   | Nodo worker                  | 20 GB    | 40 GB (/dev/vdb)     | Longhorn – almacenamiento local             |
| worker2      | 10.17.4.25   | Nodo worker                  | 20 GB    | 40 GB (/dev/vdb)     | Longhorn – almacenamiento local             |
| worker3      | 10.17.4.26   | Nodo worker                  | 20 GB    | 40 GB (/dev/vdb)     | Longhorn – almacenamiento local             |
| storage1     | 10.17.4.27   | Nodo NFS + Backups           | 10 GB    | 80 GB (/dev/vdb)     | NFS, backups, exportaciones, tokens         |
| postgresql1  | 10.17.3.14   | Base de datos externa        | 32 GB    | —                    | Cliente de almacenamiento NFS               |
| loadbalancer1| 10.17.3.12   | HAProxy + Traefik            | 32 GB    | —                    | Token JWT de Traefik                        |
| loadbalancer2| 10.17.3.13   | HAProxy + Traefik            | 32 GB    | —                    | Token JWT de Traefik                        |

---

## 🧠 Almacenamiento Total Aproximado

| Sistema      | Capacidad Total | Tipo de Acceso  | Uso Principal                             |
|--------------|------------------|------------------|--------------------------------------------|
| Longhorn     | 120 GB (3x40 GB) | ReadWriteOnce    | Volúmenes replicados (monitoring, apps)    |
| NFS          | 80 GB (LVM)      | ReadWriteMany    | PostgreSQL, carpetas compartidas, backups  |

---

## 📂 Rutas de Almacenamiento en storage1

| Ruta                    | Tamaño Aproximado | Tipo      | Uso                                         |
|-------------------------|-------------------|-----------|----------------------------------------------|
| /srv/nfs/postgresql     | 10 GB             | NFS (RWX) | Base de datos PostgreSQL                    |
| /srv/nfs/shared         | 9 GB              | NFS (RWX) | Datos compartidos entre pods                |
| /mnt/longhorn-disk      | 58 GB             | LVM       | Backups automáticos Longhorn, tokens JWT    |

Permisos: **0777**, Exportado mediante /etc/exports por Ansible.

---

## 🔀 Persistencia y Alta Disponibilidad

| Escenario                         | Comportamiento Previsto                                          |
|----------------------------------|------------------------------------------------------------------|
| Pod reiniciado                   | PVC se reatacha automáticamente                                  |
| Nodo worker caído                | Longhorn reatacha volúmenes a otro nodo disponible               |
| Reinicio de clúster              | PVCs se conservan si almacenamiento está bien configurado        |
| NFS Down                         | PVCs RWX inaccesibles (revisar tolerancias y replicas)           |

---

## 🔐 Token JWT de Traefik (Ingress Controller Externo)

- Se genera con kubectl create token traefik-sa.
- Se guarda en: /etc/traefik/token en ambos balanceadores.
- Se recomienda montar el token desde:
  - /mnt/longhorn-disk/tokens/traefik-sa.jwt
  - /srv/nfs/traefik-token (RWX compartido)

---

## 💡 Backups del clúster K3s (etcd)

- **Comando manual**:  
  k3s etcd-snapshot save --name backup-$(date +%F)

- **Ruta de almacenamiento sugerida**:  
  /mnt/longhorn-disk/backups/etcd/  
  /srv/nfs/backups/etcd/

- **Automatización por cron**:
  ```cron
  0 2 * * * /usr/local/bin/k3s etcd-snapshot save --name auto-backup-$(date +\%F) >> /var/log/etcd-backup.log 2>&1
  ```

---

## 📦 Aplicaciones que usan almacenamiento

| Aplicación            | Tipo PVC  | Sistema    | Ruta de Almacenamiento                 |
|-----------------------|-----------|------------|-----------------------------------------|
| PostgreSQL (externo)  | RWX       | NFS        | /srv/nfs/postgresql                     |
| Prometheus            | RWO       | Longhorn   | /mnt/longhorn-disk/                     |
| Grafana               | RWO       | Longhorn   | /mnt/longhorn-disk/                     |
| Elasticsearch         | RWO       | Longhorn   | /mnt/longhorn-disk/                     |
| Redis                 | RWO       | Longhorn   | /mnt/longhorn-disk/                     |
| Kafka                 | RWO       | Longhorn   | /mnt/longhorn-disk/                     |
| Nginx static assets   | RWX       | NFS        | /srv/nfs/shared/static/                 |
| Token JWT Traefik     | RWO/RWX   | Longhorn/NFS| /mnt/longhorn-disk/tokens/traefik.jwt  |

---

## 🧱 Gestión de LVM en storage1

```bash
# Volumen físico
pvcreate /dev/vdb

# Grupo de volúmenes
vgcreate vg_data /dev/vdb

# Volúmenes lógicos
lvcreate -y -L 10G -n postgresql_lv vg_data
lvcreate -y -L 9G  -n shared_lv     vg_data
lvcreate -y -L 58G -n longhorn_lv   vg_data

# Formateo y montaje
mkfs.ext4 /dev/vg_data/postgresql_lv
mkfs.ext4 /dev/vg_data/shared_lv
mkfs.ext4 /dev/vg_data/longhorn_lv

mount -a
```

---

## ✅ Buenas Prácticas

- Usa Longhorn para aplicaciones críticas con replicación.
- Usa NFS para datos compartidos (HTML, config, multimedia).
- Mantén snapshots de Longhorn automáticos y backups diarios en disco externo.
- Etiqueta los nodos de almacenamiento:  
  ```bash
  kubectl label node storage1 longhorn-node=true
  ```

---

## 📊 Visualización y Monitoreo

- **Prometheus & Grafana** usan PVCs Longhorn.
- **cAdvisor** monitorea uso de volumen por contenedor.
- **Nagios / ELK** almacenan logs en Longhorn o NFS, según configuración.

---

## 🧠 Conclusión

El diseño implementado permite:

- Almacenamiento **replicado**, **aislado por volumen**, y **resiliente a fallos**.
- Soporte para aplicaciones **stateful**, almacenamiento **compartido**, y backups internos sin depender de servicios cloud.
- Gestión 100% automatizada por Ansible, desde montaje y exportación hasta limpieza y respaldo.

---

🌟 Si el clúster se cae: tus datos están seguros, distribuidos y recuperables.
