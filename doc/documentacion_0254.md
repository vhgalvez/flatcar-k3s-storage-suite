inventory\hosts.ini
[storage]
10.17.4.27 ansible_host=10.17.4.27 ansible_user=core ansible_ssh_private_key_file=/root/.ssh/cluster_openshift/key_cluster_openshift/id_rsa_key_cluster_openshift ansible_port=22 ansible_shell_executable=/bin/sh

[masters]
10.17.4.21 ansible_host=10.17.4.21 ansible_user=core ansible_ssh_private_key_file=/root/.ssh/cluster_openshift/key_cluster_openshift/id_rsa_key_cluster_openshift ansible_port=22 ansible_shell_executable=/bin/sh
10.17.4.22 ansible_host=10.17.4.22 ansible_user=core ansible_ssh_private_key_file=/root/.ssh/cluster_openshift/key_cluster_openshift/id_rsa_key_cluster_openshift ansible_port=22 ansible_shell_executable=/bin/sh
10.17.4.23 ansible_host=10.17.4.23 ansible_user=core ansible_ssh_private_key_file=/root/.ssh/cluster_openshift/key_cluster_openshift/id_rsa_key_cluster_openshift ansible_port=22 ansible_shell_executable=/bin/sh

[workers]
10.17.4.24 ansible_host=10.17.4.24 ansible_user=core ansible_ssh_private_key_file=/root/.ssh/cluster_openshift/key_cluster_openshift/id_rsa_key_cluster_openshift ansible_port=22 ansible_shell_executable=/bin/sh
10.17.4.25 ansible_host=10.17.4.25 ansible_user=core ansible_ssh_private_key_file=/root/.ssh/cluster_openshift/key_cluster_openshift/id_rsa_key_cluster_openshift ansible_port=22 ansible_shell_executable=/bin/sh
10.17.4.26 ansible_host=10.17.4.26 ansible_user=core ansible_ssh_private_key_file=/root/.ssh/cluster_openshift/key_cluster_openshift/id_rsa_key_cluster_openshift ansible_port=22 ansible_shell_executable=/bin/sh
10.17.4.27 ansible_host=10.17.4.27 ansible_user=core ansible_ssh_private_key_file=/root/.ssh/cluster_openshift/key_cluster_openshift/id_rsa_key_cluster_openshift ansible_port=22 ansible_shell_executable=/bin/sh

[load_balancers]
10.17.3.12 ansible_host=10.17.3.12 ansible_user=core ansible_ssh_private_key_file=/root/.ssh/cluster_openshift/key_cluster_openshift/id_rsa_key_cluster_openshift ansible_port=22 ansible_shell_executable=/bin/sh
10.17.3.13 ansible_host=10.17.3.13 ansible_user=core ansible_ssh_private_key_file=/root/.ssh/cluster_openshift/key_cluster_openshift/id_rsa_key_cluster_openshift ansible_port=22 ansible_shell_executable=/bin/sh

[haproxy_keepalived]
10.17.5.10 ansible_host=10.17.5.10 ansible_user=core ansible_ssh_private_key_file=/root/.ssh/cluster_openshift/key_cluster_openshift/id_rsa_key_cluster_openshift ansible_port=22 ansible_shell_executable=/bin/sh

[all:vars]
k3s_master_ip=10.17.4.21
traefik_domain=traefik.local
letsencrypt_email=tu-email@dominio.com

roles\longhorn_worker\tasks\main.yml
roles\storage_setup\tasks\main.yml
---
- name: 🧽 Formatear disco adicional (/dev/vdb) para Longhorn
  raw: mkfs.ext4 -F /dev/vdb

- name: 📁 Crear punto de montaje para Longhorn
  raw: mkdir -p /mnt/longhorn-disk

- name: 📄 Añadir punto de montaje a /etc/fstab y montar volumen
  raw: |
    grep -q '/mnt/longhorn-disk' /etc/fstab || echo '/dev/vdb /mnt/longhorn-disk ext4 defaults 0 2' >> /etc/fstab
    mount -a

- name: 🛡️ Asignar permisos a la ruta /mnt/longhorn-disk
  raw: chmod 0777 /mnt/longhorn-disk

- name: 🏷️ Etiquetar nodo como longhorn-node
  raw: |
    nodename=$(hostname)
    kubectl label node "$nodename" longhorn-node=true --overwrite || true
  ignore_errors: true
roles\nfs_config\tasks\main.yml
---
- name: 🛠️ Activar y arrancar el servicio NFS
  raw: |
    systemctl enable --now nfs-server || true

- name: 📤 Exportar rutas NFS
  raw: |
    echo '/srv/nfs/postgresql *(rw,sync,no_subtree_check,no_root_squash)' > /etc/exports
    echo '/srv/nfs/shared *(rw,sync,no_subtree_check,no_root_squash)' >> /etc/exports
    echo '/srv/nfs/traefik-token *(rw,sync,no_subtree_check,no_root_squash)' >> /etc/exports
    exportfs -rav

- name: 🔍 Verificar exportaciones NFS activas
  raw: exportfs -v



---
- name: 🛑 Detener NFS si está activo
  raw: systemctl stop nfs-server || true
  ignore_errors: true

- name: 💽 Limpieza previa
  raw: |
    umount -f /srv/nfs/postgresql || true
    umount -f /srv/nfs/shared || true
    umount -f /mnt/longhorn-disk || true
    lvremove -fy /dev/{{ vg_name }}/postgresql_lv || true
    lvremove -fy /dev/{{ vg_name }}/shared_lv || true
    lvremove -fy /dev/{{ vg_name }}/longhorn_lv || true
    vgremove -fy {{ vg_name }} || true
    pvremove -ff -y {{ disk_device }} || true

- name: ➕ Crear LVM
  raw: |
    pvcreate -y {{ disk_device }}
    vgcreate {{ vg_name }} {{ disk_device }}
    lvcreate -y -L 10G -n postgresql_lv {{ vg_name }}
    lvcreate -y -L 9G  -n shared_lv     {{ vg_name }}
    lvcreate -y -L 58G -n longhorn_lv   {{ vg_name }}

- name: 🧱 Formatear volúmenes
  raw: |
    mkfs.ext4 -F /dev/{{ vg_name }}/postgresql_lv
    mkfs.ext4 -F /dev/{{ vg_name }}/shared_lv
    mkfs.ext4 -F /dev/{{ vg_name }}/longhorn_lv

- name: 📁 Crear puntos de montaje y configurar fstab
  raw: |
    mkdir -p /srv/nfs/postgresql /srv/nfs/shared /mnt/longhorn-disk
    grep -q postgresql /etc/fstab || echo '/dev/{{ vg_name }}/postgresql_lv /srv/nfs/postgresql ext4 defaults 0 2' >> /etc/fstab
    grep -q shared     /etc/fstab || echo '/dev/{{ vg_name }}/shared_lv     /srv/nfs/shared     ext4 defaults 0 2' >> /etc/fstab
    grep -q longhorn   /etc/fstab || echo '/dev/{{ vg_name }}/longhorn_lv   /mnt/longhorn-disk ext4 defaults 0 2' >> /etc/fstab

- name: 🔄 Montar volúmenes
  raw: mount -a

- name: 🔐 Asignar permisos a volúmenes
  raw: |
    chmod 0777 /srv/nfs/postgresql
    chmod 0777 /srv/nfs/shared
    chmod 0777 /mnt/longhorn-disk

- name: 🛠️ Activar y arrancar el servicio NFS
  raw: |
    systemctl enable --now nfs-server || true

- name: 📤 Exportar rutas NFS
  raw: |
    echo '/srv/nfs/postgresql *(rw,sync,no_subtree_check,no_root_squash)' > /etc/exports
    echo '/srv/nfs/shared *(rw,sync,no_subtree_check,no_root_squash)' >> /etc/exports
    echo '/srv/nfs/traefik-token *(rw,sync,no_subtree_check,no_root_squash)' >> /etc/exports
    exportfs -rav

- name: 🔍 Verificar exportaciones NFS activas
  raw: exportfs -v

playbook_cleanup.yml
---
- name: 🔄 Resetear almacenamiento en storage1
  hosts: storage
  become: true
  gather_facts: false

  tasks:
    - name: 🛑 Detener servicio NFS
      raw: systemctl stop nfs-server || true

    - name: 💽 Limpiar almacenamiento
      raw: |
        umount -f /srv/nfs/postgresql || true
        umount -f /srv/nfs/shared || true
        umount -f /mnt/longhorn-disk || true
        lvremove -fy /dev/vg_data/postgresql_lv || true
        lvremove -fy /dev/vg_data/shared_lv || true
        lvremove -fy /dev/vg_data/longhorn_lv || true
        vgremove -fy vg_data || true
        pvremove -ff -y /dev/vdb || true
        wipefs -a /dev/vdb
        dd if=/dev/zero of=/dev/vdb bs=1M count=10 || true

    - name: 🧹 Limpiar entradas fstab
      raw: |
        sed -i '/\/srv\/nfs\/postgresql/d' /etc/fstab
        sed -i '/\/srv\/nfs\/shared/d' /etc/fstab
        sed -i '/\/mnt\/longhorn-disk/d' /etc/fstab

    - name: 🧽 Eliminar puntos de montaje
      raw: |
        rm -rf /srv/nfs/postgresql /srv/nfs/shared /mnt/longhorn-disk

site.yml
---
- name: Configurar nodo de almacenamiento
  hosts: storage
  become: true
  gather_facts: false
  roles:
    - storage_setup 




