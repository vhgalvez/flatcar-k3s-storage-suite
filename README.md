# 📦 flatcar-k3s-storage-suite - Guía de Uso Seguro

Este proyecto Ansible proporciona playbooks seguros para configurar almacenamiento persistente en un clúster Kubernetes sobre **Flatcar Linux y Rocky Linux**, utilizando LVM, NFS y Longhorn. Las tareas han sido cuidadosamente reforzadas para evitar operaciones destructivas accidentales y garantizar una ejecución segura.

> ⚠️ **ADVERTENCIA**: Lea completamente esta guía antes de ejecutar los playbooks. Las tareas de particionado y formateo pueden eliminar datos si se usan incorrectamente.

---

## ⚙️ Componentes incluidos

- Configuración de volúmenes LVM para:
  - PostgreSQL (`/srv/nfs/postgresql`)
  - Datos compartidos (`/srv/nfs/shared`)
  - Longhorn (`/mnt/longhorn-disk`)
- Exportación NFS (automática)
- Preparación automática y segura de discos `/dev/vdb`
- Playbook de limpieza con confirmación obligatoria

---

## ⚠️ Recomendaciones de Seguridad

- **Inventario validado:** Revise `inventory/hosts.ini`. Asegúrese de que solo nodos con discos secundarios estén en los grupos `storage` o `longhorn_nodes`.
- **Evitar nodos críticos:** **NO incluya** en estos grupos a los nodos master ni al servidor de virtualización.
- **Discos secundarios solamente:** Solo se operará sobre `/dev/vdb`. **No se tocarán discos del sistema como `/dev/vda`, `/dev/sda`, `nvme0n1`, etc.**
- **Confirmación para limpieza:** El playbook de limpieza solo se ejecutará si pasa la variable `confirm_cleanup=yes`.

---

## ✅ Ejecución Segura - Paso a Paso

### 1. Configurar acceso SSH
Configure acceso mediante clave SSH al usuario `core` desde el nodo de control hacia todos los nodos del clúster.

### 2. Verificar inventario (`inventory/hosts.ini`)
Confirme que los nodos estén correctamente agrupados:
- Grupo `storage`: nodos con volúmenes LVM y NFS (ej. `10.17.4.27`)
- Grupo `workers`: nodos con disco para Longhorn (`10.17.4.24`, `10.17.4.25`, etc.)

### 3. Ejecutar configuración de almacenamiento
```bash
sudo ansible-playbook -i inventory/hosts.ini site.yml
```
Este playbook ejecuta:
- **Detección y validación de discos**: Solo se utiliza `/dev/vdb` si está completamente libre.
- **Particionado y creación de LVM**: Se crean volúmenes lógicos para PostgreSQL, compartidos y Longhorn.
- **Formateo y montaje**: Se usan sistemas de archivos `ext4` y se montan en directorios adecuados.
- **Instalación de NFS**: Se instala `nfs-server` y se configura `/etc/exports` con las rutas compartidas.
- **Habilitación del servicio**: Se habilita y arranca el servicio NFS automáticamente.

### 4. Validar configuración de discos montados
En `storage1`, ejecute:
```bash
df -h
```
Verifique que aparecen montados:
- `/srv/nfs/postgresql`
- `/srv/nfs/shared`
- `/mnt/longhorn-disk`

Desde Longhorn, compruebe que los nodos usan `/mnt/longhorn-disk` como almacenamiento.

---

## 📘 Tareas y su descripción

### 🧱 `storage_setup` (rol)
1. **Detección segura del disco**: Verifica que `/dev/vdb` exista y esté sin uso. Evita accidentalmente tocar discos críticos.
2. **Particionado y LVM**: Crea una partición primaria, grupo de volúmenes (`vg_storage`), y tres volúmenes lógicos.
3. **Formateo y montaje**: Usa `ext4` para los volúmenes y los monta en las rutas configuradas.
4. **Exportación NFS**: Crea `exports` para PostgreSQL y datos compartidos, arranca `nfs-server`.
5. **Reutilizable**: Si el disco ya está formateado, las tareas se saltan para evitar sobrescritura.

### 💾 `longhorn_worker` (rol)
1. **Verificación del disco**: Comprueba que `/dev/vdb` no tenga particiones, FS ni montajes.
2. **Formateo y montaje**: Si está libre, se formatea y monta en `/mnt/longhorn-disk`.
3. **Seguro**: Si el disco ya tiene uso, no se hace nada.

### 🚀 `install_longhorn.yml` (playbook)
1. **Etiquetado de nodos**: Etiqueta nodos `worker` con `longhorn-node=true`.
2. **Instalación Longhorn**: Crea el namespace, descarga y aplica el manifiesto oficial.
3. **Espera por pods**: Valida que los pods estén `Ready`.
4. **Resumen final**: Muestra estado de los pods Longhorn.

### 🧹 `playbook_cleanup.yml` (playbook opcional)
1. **Verificación de seguridad**: Requiere `-e confirm_cleanup=yes` para ejecutarse.
2. **Detiene NFS**: Apaga el servicio si está corriendo.
3. **Desmontaje de volúmenes**: Libera rutas montadas.
4. **Elimina LVM y particiones**: Borra volúmenes, VG, particiones y firma LVM.
5. **No afecta VMs**: Seguro para reprovisionar nodos sin borrar la máquina virtual.

---

## 🧩 Estado de Discos

### 🔧 Nodo worker1
- `vda`: 20G (SO, montado en `/`)
- `vdb`: 40G (libre, sin formatear)

### 🗃️ Nodo storage1
- `vda`: 10G (SO, montado en `/`)
- `vdb`: 80G (libre, sin uso, listo para NFS o LVM)

### ✅ Resumen Final

| Nodo      | Disco SO (vda) | Uso Sistema | Disco Adicional (vdb) | Estado Disco |
|-----------|----------------|-------------|------------------------|---------------|
| worker1   | 20 GB          | ~2.5 GB     | 40 GB                  | Libre         |
| storage1  | 10 GB          | ~2.5 GB     | 80 GB                  | Libre         |

---

## 🧹 Limpieza del nodo de almacenamiento (opcional)

Si necesitas reiniciar desde cero el nodo `storage` (por ejemplo, para reprovisionarlo sin destruir la VM), puedes ejecutar:

```bash
sudo ansible-playbook playbooks/playbook_cleanup.yml -e confirm_cleanup=yes
```

Este playbook:
- Desmonta volúmenes
- Elimina LVM (vg y lvs)
- Borra la partición y firma de LVM de `/dev/vdb`
- Detiene el servicio NFS

⚠️ **Usar solo si sabes lo que estás haciendo. No borra VMs.**

---

## 🛡️ Conclusión

Este conjunto de playbooks garantiza una configuración de almacenamiento automatizada y segura para su clúster Kubernetes con Flatcar. Gracias a las validaciones y protecciones incluidas, puede trabajar con confianza evitando daños accidentales al sistema operativo o pérdida de datos.

> **Repositorio del proyecto:** [`flatcar-k3s-storage-suite`](https://github.com/tu_usuario/flatcar-k3s-storage-suite)