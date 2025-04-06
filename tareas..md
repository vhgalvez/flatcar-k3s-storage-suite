# 📦 flatcar-k3s-storage-suite – README

Automatiza la preparación de almacenamiento persistente para un clúster Kubernetes con K3s y Flatcar Linux. Este proyecto de Ansible está diseñado para entornos **bare-metal o virtualizados**, y permite usar tanto **Longhorn** (volúmenes replicados) como **NFS** (datos compartidos) de forma complementaria.

---

## ✅ Checklist de Automatización

### 🗂️ storage1 – Configuración del nodo de almacenamiento (LVM + NFS)

#### Volúmenes LVM
- [ ] Verificar existencia del disco adicional `/dev/vdb`
- [ ] Crear volumen físico con `pvcreate`
- [ ] Crear grupo de volúmenes `vg_data`
- [ ] Crear volúmenes lógicos:
  - [ ] `postgresql_lv` (10 GB)
  - [ ] `shared_lv` (9 GB)
  - [ ] `longhorn_lv` (58 GB)
- [ ] Formatear volúmenes `mkfs.ext4`
- [ ] Crear puntos de montaje:
  - [ ] `/srv/nfs/postgresql`
  - [ ] `/srv/nfs/shared`
  - [ ] `/mnt/longhorn-disk`
- [ ] Establecer permisos `0777` en los puntos montados
- [ ] Agregar entradas en `/etc/fstab`
- [ ] Montar con `mount -a`

#### NFS Server
- [ ] Instalar y habilitar `nfs-server`
- [ ] Configurar `/etc/exports`:
  - [ ] `/srv/nfs/postgresql`
  - [ ] `/srv/nfs/shared`
  - [ ] `/srv/nfs/traefik-token` (opcional)
- [ ] Aplicar cambios con `exportfs -ra`

---

### 💽 worker1/2/3 – Preparar discos para Longhorn (Flatcar Linux)

- [ ] Verificar existencia de `/dev/vdb`
- [ ] Formatear disco como `ext4` (sin particionar)
- [ ] Crear punto de montaje `/var/lib/longhorn`
- [ ] Agregar a `/etc/fstab`
- [ ] Montar disco
- [ ] Crear etiqueta en nodo:
  ```bash
  kubectl label node <worker-node> longhorn-node=true
  ```

---

### 🔐 Token JWT de Traefik

- [ ] Crear directorio `/mnt/longhorn-disk/tokens/`
- [ ] Copiar token JWT de Traefik
- [ ] Exportar también a `/srv/nfs/traefik-token` (opcional)

---

### 💡 Backups del clúster K3s (etcd)

- [ ] Crear carpetas:
  - [ ] `/mnt/longhorn-disk/backups/etcd/`
  - [ ] `/srv/nfs/backups/etcd/`
- [ ] Configurar `cron` diario:
  ```cron
  0 2 * * * /usr/local/bin/k3s etcd-snapshot save --name auto-backup-$(date +\%F) >> /var/log/etcd-backup.log 2>&1
  ```

---

### 🧼 Playbook de limpieza (`playbook_cleanup.yml`)

- [ ] Desmontar volúmenes en `storage1` y `workers`
- [ ] Eliminar entradas del `fstab`
- [ ] Remover volúmenes lógicos (`lvremove`)
- [ ] Remover grupo de volúmenes (`vgremove`)
- [ ] Remover volumen físico (`pvremove`)
- [ ] Limpiar `/etc/exports` (si aplica)
- [ ] Detener `nfs-server`

---

### 🧱 Kubernetes – Integración opcional

- [ ] Crear `StorageClass` para Longhorn
- [ ] Crear PVCs para apps críticas (Prometheus, Redis, etc.)
- [ ] Validar funcionamiento de PVCs RWO (Longhorn) y RWX (NFS)

---

## 📁 Estructura del Proyecto

- `site.yml`: orquestación principal
- `nfs_config.yml`: configuración de NFS
- `playbook_cleanup.yml`: limpieza total
- `inventory/hosts.ini`: inventario completo
- `roles/storage_setup/`: lógica para `storage1`
- `roles/longhorn_worker/`: lógica para workers
- Variables embebidas directamente en `tasks/main.yml`

---

## 🚀 Ejecución

```bash
# Setup completo
ansible-playbook -i inventory/hosts.ini site.yml

# Exportar NFS
ansible-playbook -i inventory/hosts.ini nfs_config.yml

# (Opcional) Limpieza
ansible-playbook -i inventory/hosts.ini playbook_cleanup.yml
```

---

🎯 Diseñado para funcionar con el clúster Kubernetes HA de `FlatcarMicroCloud`


flatcar-k3s-storage-suite/
├── inventory/
│   └── hosts.ini
│
├── roles/
│   ├── storage_setup/
│   │   └── tasks/
│   │       └── main.yml
│   │
│   ├── longhorn_worker/
│   │   └── tasks/
│   │       └── main.yml
│   │
│   └── nfs_config/
│       └── tasks/
│           └── main.yml
│
├── site.yml
├── nfs_config.yml
├── playbook_cleanup.yml
└── README.md