# flatcar-k3s-longhorn-workers

## 📦 Almacenamiento en Workers (`worker1`, `worker2`, `worker3`)

Estos tres nodos están dedicados al **almacenamiento persistente replicado con Longhorn**, utilizando un **disco adicional** en cada nodo para almacenar volúmenes `RWO` (ReadWriteOnce).

---

## 🧱 Características Comunes

| Atributo                  | Valor                           |
|---------------------------|----------------------------------|
| Nodos                     | `worker1`, `worker2`, `worker3`  |
| IPs                       | `10.17.4.24 – 10.17.4.26`         |
| Disco adicional           | `/dev/vdb` (40 GB)               |
| Ruta de montaje           | `/mnt/longhorn-disk`            |
| Tipo de volumen           | Local `ext4` montado con Ansible |
| Acceso                    | `ReadWriteOnce` (RWO)            |
| Etiqueta Kubernetes       | `longhorn-node=true`             |
| Proveedor de almacenamiento | Longhorn                       |

---

## 📁 Estructura

En cada nodo worker:

```bash
/dev/vdb                ← Disco adicional sin particionar
└→ Formateado en ext4
└→ Montado en: /mnt/longhorn-disk
└→ Anotado en: /etc/fstab
└→ Permisos: 0777
```

---

## 💾 Volúmenes posibles a ubicarse aquí

- `Prometheus`: `/mnt/longhorn-disk/prometheus/`
- `Grafana`: `/mnt/longhorn-disk/grafana/`
- `Redis`: `/mnt/longhorn-disk/redis/`
- `Kafka`: `/mnt/longhorn-disk/kafka/`
- `Elasticsearch`: `/mnt/longhorn-disk/elasticsearch/`
- `Tokens Traefik (opcional)`: `/mnt/longhorn-disk/tokens/`

---

Este repositorio automatiza esta configuración mediante Ansible para garantizar consistencia, eficiencia y escalabilidad en entornos Kubernetes sobre Flatcar Linux.

longhorn-worker-storage-setup/
├── inventory/
│   └── hosts.ini                   # Inventario con los nodos worker
├── roles/
│   └── longhorn_worker/
│       └── tasks/
│           └── main.yml           # Lógica para formatear, montar y etiquetar
├── site.yml                       # Playbook principal con rol
├── playbook.yml                   # (Opcional) Playbook sin rol, todo en uno
└── README.md                      # Documentación del proyecto
