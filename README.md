# 📦 Ansible Storage Cluster - FlatcarMicroCloud

Este proyecto Ansible automatiza la configuración de un nodo de almacenamiento (`storage1`) en un entorno Kubernetes, especialmente diseñado para **Flatcar Linux** y entornos sin Python. Se utiliza principalmente como parte del proyecto [FlatcarMicroCloud](https://github.com/vhgalvez/FlatcarMicroCloud).

## 🚨 Advertencia Importante

> ⚠️ **Este playbook destruye y recrea todos los volúmenes lógicos (LVM) en el disco `/dev/vdb`.**
>
> Si ya existen volúmenes con datos importantes en este disco, **haz una copia de seguridad antes de ejecutar** este código.
>
> Este playbook **está pensado para ejecutarse en entornos desde cero**, típicamente recién provisionados con Terraform para laboratorios y homelabs.

---

## 🧱 Qué hace este Playbook

Este playbook realiza lo siguiente sobre el nodo `storage1`:

1. Elimina cualquier volumen lógico, grupo de volúmenes o particiones existentes en `/dev/vdb`.
2. Crea un nuevo volumen físico y grupo de volúmenes (`vg_data`) sobre `/dev/vdb`.
3. Crea los siguientes volúmenes lógicos:
   - `postgresql_lv` → montado en `/srv/nfs/postgresql` (10 GB)
   - `shared_lv` → montado en `/srv/nfs/shared` (10 GB)
   - `longhorn_lv` → montado en `/mnt/longhorn-disk` (60 GB)
4. Instala y configura el servidor NFS.
5. Exporta los volúmenes NFS para que puedan ser montados por otros nodos.
6. Etiqueta el nodo como compatible con Longhorn.

---

## 📂 Estructura del Proyecto

```bash
ansible-storage-cluster/
├── inventory/
│   └── hosts.ini               # Inventario con IP del nodo storage1
├── roles/
│   ├── lvm_setup/              # Tareas para preparar LVM
│   ├── nfs_server/             # Instalación y configuración de NFS
│   └── longhorn_node/          # Etiquetado del nodo para Longhorn
├── site.yml                    # Playbook principal
└── README.md                   # Este archivo
```
⚙️ Requisitos Previos
Servidor o VM con Flatcar Linux

Disco /dev/vdb vacío de 80 GB para almacenamiento

Acceso SSH con clave privada configurada en inventory/hosts.ini

Ansible 2.14+ instalado en el nodo controlador

🚀 Ejecución
Clona el repositorio:

bash
Copiar
Editar
git clone https://github.com/vhgalvez/ansible-storage-cluster.git
cd ansible-storage-cluster
Edita tu inventario:

Verifica que inventory/hosts.ini tenga la IP correcta y el usuario adecuado para el nodo storage1.

Ejecuta el playbook:

bash
Copiar
Editar
sudo ansible-playbook -i inventory/hosts.ini site.yml
📌 Resultado Esperado
Una vez completado, el nodo storage1 tendrá:

Punto de Montaje	Tamaño	Uso
/srv/nfs/postgresql	10 GB	Datos de PostgreSQL vía NFS
/srv/nfs/shared	10 GB	Datos compartidos RWX en pods
/mnt/longhorn-disk	60 GB	Volúmenes distribuidos Longhorn
🧪 Uso en conjunto con Terraform
Este Ansible está pensado para ejecutarse después de provisionar las VMs con el proyecto Terraform de red nat_network_03, que define el disco de 80 GB (/dev/vdb) en storage1.

✍️ Autor
vhgalvez
🔗 GitHub | 🧠 Proyecto: FlatcarMicroCloud

🛡️ Licencia
MIT License — Puedes usarlo libremente con fines educativos o personales.




