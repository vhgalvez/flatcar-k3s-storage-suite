# 📦 Ansible Storage Cluster – FlatcarMicroCloud

Este proyecto automatiza la configuración de un nodo de almacenamiento (`storage1`) usando **Ansible**, optimizado para **Flatcar Linux** (sin Python) y clústeres Kubernetes con **K3s**. Forma parte del ecosistema [FlatcarMicroCloud](https://github.com/vhgalvez/FlatcarMicroCloud).

---

## ✨ Visión General

El nodo `storage1` ofrece almacenamiento persistente y distribuido de alta disponibilidad para:

- Bases de datos como **PostgreSQL**
- Datos compartidos entre pods (**RWX**)
- Volúmenes gestionados por **Longhorn (RWO)**

Utiliza **LVM**, **NFS** y almacenamiento local en `/dev/vdb`.

---

## 📚 Tabla de Roles y Almacenamiento por Nodo

| Nodo        | Disco                 | Rol en Longhorn                    | Observaciones                            |
|-------------|-----------------------|------------------------------------|------------------------------------------|
| `storage1`  | `/mnt/longhorn-disk`  | Nodo **dedicado** de almacenamiento | ✅ Ideal: marcar como `Not Schedulable` |
| `worker1`   | Disco local de 50 GB   | Nodo mixto (cálculo + almacenamiento) | ✅ Recomendado                         |
| `worker2`   | Disco local de 50 GB   | Nodo mixto                          | ✅                                       |
| `worker3`   | Disco local de 50 GB   | Nodo mixto                          | ✅                                       |

> ⚠️ Se recomienda marcar `storage1` como **Not Schedulable** en Longhorn para evitar ejecutar pods ahí.

---

## 📁 Directorios Montados en `storage1`

| Ruta                    | Propósito                                  | Tipo de Acceso     |
|-------------------------|--------------------------------------------|--------------------|
| `/srv/nfs/postgresql`   | Volumen persistente para PostgreSQL vía NFS | RW (Read/Write)    |
| `/srv/nfs/shared`       | Volumen compartido para pods                | RWX (ReadWriteMany)|
| `/mnt/longhorn-disk`    | Disco para almacenamiento Longhorn          | RWO (ReadWriteOnce)|

---

## 🔧 Tecnologías Usadas

- **LVM**: Para crear volúmenes lógicos escalables sobre `/dev/vdb`
- **NFS Server**: Exporta volúmenes accesibles por otros nodos
- **Longhorn**: Almacenamiento distribuido para Kubernetes

---

## ⚙️ Requisitos Previos

- Nodo o VM con **Flatcar Linux**
- Disco adicional (`/dev/vdb`) de **80 GB**
- Acceso SSH con clave en `inventory/hosts.ini`
- Ansible 2.14+ instalado en el nodo controlador

---

## 📂 Estructura del Proyecto

```bash
ansible-storage-cluster/
├── inventory/
│   └── hosts.ini               # Inventario con IP del nodo storage1
├── roles/
│   ├── lvm_setup/              # Configura volúmenes LVM
│   ├── nfs_server/             # Montaje y formateo de rutas NFS
│   ├── nfs_config/             # Exportación y activación de NFS
│   └── longhorn_node/          # Marca nodo como apto para Longhorn
├── site.yml                    # Playbook principal
├── nfs_config.yml              # Playbook adicional: exportación NFS
└── README.md                   # Este archivo
   
```

🚀 Ejecución

1️⃣ Configurar almacenamiento (`/dev/vdb`)

```bash
sudo ansible-playbook -i inventory/hosts.ini site.yml
```

Esto configura LVM, crea puntos de montaje y prepara los volúmenes para NFS y Longhorn.

2️⃣ Exportar rutas NFS y activar servicio

```bash
sudo ansible-playbook -i inventory/hosts.ini nfs_config.yml
```

Esto asegura que `/etc/exports` esté correctamente configurado y que el servidor NFS esté activo.

📌 Resultado Esperado

Punto de Montaje	Tamaño	Uso
/srv/nfs/postgresql	10 GB	Datos de PostgreSQL vía NFS
/srv/nfs/shared	10 GB	Datos compartidos RWX en pods
/mnt/longhorn-disk	60 GB	Volúmenes distribuidos Longhorn


🧪 Verificación

🔍 Comprobar volúmenes montados
```bash
df -h
```

🔍 Ver exportaciones NFS

```bash

sudo exportfs -v
```

Deberías ver:

```bash
/srv/nfs/postgresql  *(rw,sync,no_subtree_check,no_root_squash)
/srv/nfs/shared      *(rw,sync,no_subtree_check,no_root_squash)
```

🔍 Estado del servicio NFS

```bash
systemctl status nfs-server
```

🧷 Montar NFS desde otro nodo (por ejemplo, postgresql1)

```bash
sudo mount -t nfs storage1.cefaslocalserver.com:/srv/nfs/postgresql /mnt
```

🌟 Conclusión

La arquitectura resultante:

✅ Separa la carga de cómputo y el almacenamiento

✅ Usa volúmenes tolerantes a fallos con Longhorn (3 réplicas)

✅ Soporta PostgreSQL, Prometheus, Grafana, microservicios, etc.

✅ Está lista para escalar horizontalmente y de forma segura

Ideal para entornos educativos, laboratorios o preproducción realistas.

✍️ Autor

vhgalvez

🔗 FlatcarMicroCloud

🛡️ Licencia
MIT License — Libre para uso educativo y personal.