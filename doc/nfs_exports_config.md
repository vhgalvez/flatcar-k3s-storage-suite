# 📦 Configuración NFS para Kubernetes (FlatcarMicroCloud)

Este documento describe cómo configurar correctamente las exportaciones NFS en el nodo de almacenamiento (`storage1`) para que los diferentes servicios del clúster Kubernetes tengan acceso seguro y funcional a los volúmenes compartidos.

---

## 📁 Archivo `/etc/exports`

```bash
/srv/nfs/postgresql        *(rw,sync,no_subtree_check,no_root_squash)
/srv/nfs/shared            *(rw,sync,no_subtree_check,no_root_squash)
/srv/nfs/traefik-token     10.17.3.0/24(rw,sync,no_subtree_check,no_root_squash) \
                           10.17.4.0/24(rw,sync,no_subtree_check,no_root_squash) \
                           10.17.5.0/24(rw,sync,no_subtree_check,no_root_squash) \
                           192.168.0.0/24(rw,sync,no_subtree_check,no_root_squash)
```

---

## ✅ Explicación por sección

### `/srv/nfs/postgresql`
🔹 Permite que **cualquier IP** pueda montar este directorio.  
✔️ Útil en entornos de laboratorio donde no se requiere control estricto.

### `/srv/nfs/shared`
🔹 Similar al anterior, permite acceso completo a los pods con volúmenes **RWX (ReadWriteMany)**.  
✔️ Ideal para compartir archivos entre microservicios.

### `/srv/nfs/traefik-token`
🔐 Este directorio contiene el token JWT para Traefik. Aquí restringimos el acceso solo a subredes específicas:

- `10.17.3.0/24`: 🔄 Load Balancers (nodos con Traefik)
- `10.17.4.0/24`: 🧠 Masters, Workers y nodo Storage
- `10.17.5.0/24`: 💡 IPs Virtuales (VIPs - HAProxy/Keepalived)
- `192.168.0.0/24`: 🛡 Red local (pfSense, gestión interna)

---

## 🔄 Recargar la configuración NFS

Después de modificar el archivo `/etc/exports`, recarga la configuración con:

```bash
sudo exportfs -rav
sudo systemctl restart nfs-server
```

✅ Esto asegura que los nuevos permisos estén activos y visibles para todos los nodos del clúster.
