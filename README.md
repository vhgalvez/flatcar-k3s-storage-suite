# flatcar-k3s-storage-suite - Guía de Uso Seguro

Este proyecto Ansible proporciona playbooks seguros para configurar almacenamiento persistente en un clúster Kubernetes sobre **Flatcar Linux y Rocky Linux**, utilizando LVM, NFS y Longhorn. Las tareas han sido cuidadosamente reforzadas para evitar operaciones destructivas accidentales y garantizar una ejecución segura.

> ⚠️ **ADVERTENCIA**: Lea completamente esta guía antes de ejecutar los playbooks. Las tareas de particionado y formateo pueden eliminar datos si se usan incorrectamente.

---

## ⚙️ Componentes incluidos

- Configuración de volúmenes LVM para:
  - PostgreSQL (`/srv/nfs/postgresql`)
  - Datos compartidos (`/srv/nfs/shared`)
  - Longhorn (`/mnt/longhorn-disk`)
- Exportación NFS (opcional)
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
Asegúrese de tener acceso sin contraseña (mediante clave) a todos los nodos, usando el usuario `core` y la clave privada indicada en el inventario.

### 2. Verificar inventario (`inventory/hosts.ini`)
Confirme que los nodos estén correctamente agrupados:
- Grupo `storage`: nodos con volúmenes LVM y NFS (ej. `10.17.4.27`)
- Grupo `longhorn_nodes`: nodos con disco para Longhorn (`10.17.4.24`, `10.17.4.27`)
- No incluir aquí `master1`, `master2`, `master3` ni el servidor de virtualización

### 3. Ejecutar configuración de almacenamiento
```bash
sudo ansible-playbook -i inventory/hosts.ini site.yml
```

Este playbook:
- Detecta y valida que `/dev/vdb` esté presente, vacío y sin montar
- Crea volúmenes LVM para PostgreSQL, datos compartidos y Longhorn
- Formatea los volúmenes y los monta en rutas adecuadas

---

### 4. Configurar NFS (si se requiere)
Ejecute el siguiente playbook en el nodo de almacenamiento (ej. `storage1`) después de `site.yml`:
```bash
sudo ansible-playbook -i inventory/hosts.ini nfs_config.yml
```

Esto activará el servicio NFS y exportará los directorios necesarios.

---

### 5. Validar configuración
En `storage1`, ejecute:
```bash
df -h
```
Verifique que aparecen montados:
- `/srv/nfs/postgresql`
- `/srv/nfs/shared`
- `/mnt/longhorn-disk`

En Longhorn (UI o `kubectl`), verifique que los nodos usan `/mnt/longhorn-disk` correctamente.

---

## 🧹 Limpieza (solo si desea borrar todo)

⚠️ **Advertencia:** Esto eliminará todos los datos del disco `/dev/vdb`. Úselo solo si desea reprovisionar el nodo desde cero.

```bash
sudo ansible-playbook -i inventory/hosts.ini playbook_cleanup.yml -e "confirm_cleanup=yes"
```

Este playbook desmontará los volúmenes, eliminará los LVs, el VG, y la partición de `/dev/vdb`, **solo si confirma la limpieza**.

---

## 🔍 Notas Finales

- **Idempotencia:** El playbook es seguro para ejecutarse múltiples veces. Si el disco ya está configurado, abortará o saltará las tareas.
- **Extensibilidad:** Puede añadir más nodos con discos adicionales en los grupos `storage` o `longhorn_nodes` sin modificar los roles.
- **Monitoreo:** Verifique periódicamente espacio libre y salud de discos. Incluya los puntos de montaje en su sistema de backups.

---

## 🛡️ Conclusión

Este conjunto de playbooks garantiza una configuración de almacenamiento automatizada y segura para su clúster Kubernetes con Flatcar. Gracias a las validaciones y protecciones incluidas, puede trabajar con confianza evitando daños accidentales al sistema operativo o pérdida de datos.

> **Repositorio del proyecto:** [`flatcar-k3s-storage-suite`](https://github.com/tu_usuario/flatcar-k3s-storage-suite)


🔧 Nodo worker1
📦 Discos:
bash
Copiar
Editar
vda  -> 20G (Disco del sistema operativo)
vdb  -> 40G (Disco adicional)
📂 Sistema operativo (montado en /dev/vda9):
/: 17G (montado en vda9)

Espacio usado: 2.4G

Espacio libre: 14G

📊 Uso total de disco (SO):
Total ocupado por el sistema (aproximadamente): 2.4G en raíz + mínimo uso en /usr y /oem

Total ocupado ≈ 2.5 GB

💾 Disco adicional (vdb):
Tamaño: 40 GB

No está montado ni usado (aún disponible para LVM, Longhorn, NFS, etc.)

🗃️ Nodo storage1
📦 Discos:
bash
Copiar
Editar
vda  -> 10G (Disco del sistema operativo)
vdb  -> 80G (Disco adicional)
📂 Sistema operativo (montado en /dev/vda9):
/: 7.2G

Espacio usado: 2.4G

Espacio libre: 4.5G

📊 Uso total de disco (SO):
Total ocupado por el sistema (aproximadamente): 2.4 GB + mínimo uso en /usr y /oem

Total ocupado ≈ 2.5 GB

💾 Disco adicional (vdb):
Tamaño: 80 GB

No está montado ni usado (puedes usarlo para NFS, backups, etc.)

✅ Resumen Final
Nodo	Disco SO (vda)	Uso Sistema	Disco Adicional (vdb)	Estado Disco Adicional
worker1	20 GB	~2.5 GB	40 GB	Libre
storage1	10 GB	~2.5 GB	80 GB	Libre
