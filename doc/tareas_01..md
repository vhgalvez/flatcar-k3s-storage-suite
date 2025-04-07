
💾 6. Aprovisionamiento del Almacenamiento Persistente (Longhorn + NFS)

Configura el almacenamiento persistente del clúster usando NFS y Longhorn mediante el proyecto flatcar-k3s-storage-suite. Este paso prepara los discos, volúmenes LVM, exportaciones NFS y etiquetas para Longhorn, todo automatizado con Ansible.

Repositorio:
flatcar-k3s-storage-suite

Pasos:
bash
Copiar
Editar
# Clona el proyecto de almacenamiento
sudo git clone https://github.com/vhgalvez/flatcar-k3s-storage-suite.git
cd flatcar-k3s-storage-suite

# Ejecutar playbook completo (workers + storage)
ansible-playbook -i inventory/hosts.ini site.yml

# Exportar rutas NFS compartidas

ansible-playbook -i inventory/hosts.ini nfs_config.yml
💡 Recomendación: En esta fase también puedes crear tus PVCs base para Prometheus, Grafana, PostgreSQL, Redis, etc., incluso si aún no has instalado las apps. Así preparas el terreno para el paso 7.