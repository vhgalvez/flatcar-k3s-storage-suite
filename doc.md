# Arquitectura de Almacenamiento – FlatcarMicroCloud

## ✨ Resumen General

Este documento describe detalladamente la arquitectura de almacenamiento implementada en el proyecto **FlatcarMicroCloud**, un clúster Kubernetes optimizado sobre servidores físicos y máquinas virtuales con **K3s + Longhorn + NFS**.

---

## 📊 Objetivo

Garantizar almacenamiento persistente, distribuido y tolerante a fallos para:

- Bases de datos como **PostgreSQL**
- Herramientas de monitoreo como **Prometheus** y **Grafana**
- Microservicios y aplicaciones con requerimientos de volumen **RWO/RWX**

---

## 🔄 Topología y Distribución de Volúmenes

Cuando se configuran **volúmenes con 3 réplicas** en Longhorn y existen **3 nodos Worker**, se distribuye 1 réplica por nodo. Esto proporciona:

- Alta disponibilidad
- Balanceo de carga
- Recuperación automática ante fallos

---

## 📚 Tabla de Roles y Almacenamiento por Nodo

| Nodo        | Disco                   | Rol en Longhorn                    | Observaciones                              |
|-------------|-------------------------|------------------------------------|--------------------------------------------|
| `storage1`  | `/mnt/longhorn-disk`    | Nodo **dedicado** de almacenamiento | ✅ Ideal: marcar como `Not Schedulable`   |
| `worker1`   | Disco local de 50 GB     | Nodo mixto (cálculo + almacenamiento) | ✅ Recomendado                           |
| `worker2`   | Disco local de 50 GB     | Nodo mixto (cálculo + almacenamiento) | ✅                                       |
| `worker3`   | Disco local de 50 GB     | Nodo mixto (cálculo + almacenamiento) | ✅                                       |

> ⚠️ Se recomienda configurar `storage1` como `Not Schedulable` en Longhorn para evitar ejecución de pods de aplicación.

---

## 📁 Directorios Montados en `storage1`

| Ruta                    | Propósito                                  | Tipo de Acceso     |
|-------------------------|--------------------------------------------|--------------------|
| `/srv/nfs/postgresql`   | Volumen persistente para PostgreSQL via NFS | RW (Read/Write)    |
| `/srv/nfs/shared`       | Volumen compartido RWX para pods           | RWX (ReadWriteMany)|
| `/mnt/longhorn-disk`    | Disco dedicado a Longhorn (backend RWO)    | RWO (ReadWriteOnce)|

---

## 🛠️ Tecnologías Usadas

- **LVM**: Para crear volúmenes lógicos separados y escalables
- **NFS Server**: Exportación de volúmenes para PostgreSQL y datos compartidos
- **Longhorn**: Almacenamiento distribuido para Kubernetes con snapshot, backup y auto-healing

---

## 🔧 Recomendaciones Adicionales

- Activar **Replica Soft Anti-Affinity** en Longhorn para tolerancia a fallos de nodos
- Usar **Backing Images** para clonar rápidamente aplicaciones base
- Configurar **Respaldo Longhorn** en volumenes sensibles (ej. Prometheus, PostgreSQL)
- Monitorizar salud del almacenamiento con **Grafana + Prometheus**

---

## 🌟 Conclusión

La arquitectura propuesta es **sólida, flexible y escalable**, adecuada para entornos reales de desarrollo o preproducción.

Separa claramente:

- 🚀 La computación (Worker nodes)
- 📀 El almacenamiento (Nodo dedicado `storage1`)

Y se adapta a distintos tipos de volúmenes (RW, RWX, RWO) sin dificultad.

---

✅ **Todo está listo para escalar tu clúster y asegurar tus datos de forma profesional.**

