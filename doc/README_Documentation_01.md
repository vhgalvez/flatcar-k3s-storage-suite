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

## 🧼 Playbook de limpieza (`playbook_cleanup.yml`)
- Desmontar volúmenes en storage1 y workers
- Eliminar entradas del fstab
- Remover volúmenes lógicos (lvremove)
- Remover grupo de volúmenes (vgremove)
- Remover volumen físico (pvremove)
- Limpiar /etc/exports (si aplica)
- Detener nfs-server

---

## 📁 Estructura del Proyecto
```
flatcar-k3s-storage-suite/
├── inventory/
│   └── hosts.ini
├── roles/
│   ├── storage_setup/
│   │   └── tasks/
│   │       └── main.yml
│   ├── longhorn_worker/
│   │   └── tasks/
│   │       └── main.yml
│   └── nfs_config/
│       └── tasks/
│           └── main.yml
├── site.yml
├── nfs_config.yml
├── playbook_cleanup.yml
└── README.md
```

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

## 📜 Descripción de Archivos


```plaintext
flatcar-k3s-storage-suite/
├── inventory/
│   └── hosts.ini               # Inventario de nodos
├── roles/
│   ├── storage_setup/          # Configura LVM + NFS en storage1
│   │   └── tasks/main.yml
│   ├── longhorn_worker/        # Configura disco adicional en workers
│   │   └── tasks/main.yml
│   └── nfs_config/             # Configura exportaciones NFS
│       └── tasks/main.yml
├── site.yml                    # Orquesta todo en storage1
├── nfs_config.yml              # (Independiente) Exportar rutas NFS
├── playbook_cleanup.yml        # Limpieza completa de almacenamiento
└── README.md                   # Documentación general del proyecto
```

