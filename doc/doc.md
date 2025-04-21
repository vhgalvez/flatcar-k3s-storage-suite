
kubectl delete -f https://raw.githubusercontent.com/longhorn/longhorn/v1.6.1/deploy/longhorn.yaml

kubectl delete pods --all -n longhorn-system
kubectl delete deployments --all -n longhorn-system
kubectl delete daemonsets --all -n longhorn-system
kubectl delete statefulsets --all -n longhorn-system
kubectl delete svc --all -n longhorn-system
kubectl delete pvc --all -n longhorn-system

kubectl delete namespace longhorn-system
kubectl delete crd $(kubectl get crd | grep longhorn.io | awk '{print $1}')

kubectl delete pv -l app=longhorn
kubectl delete pvc -l app=longhorn




lsblk
sudo pvscan
sudo vgscan
sudo lvscan


kubectl get pods -n longhorn-system
kubectl get svc -n longhorn-system
kubectl get pv
kubectl get pvc

sudo lvremove -y /dev/vg_storage/postgres_lv
sudo lvremove -y /dev/vg_storage/shared_lv
sudo lvremove -y /dev/vg_storage/longhorn_lv
sudo vgremove -y vg_storage
sudo pvremove -y /dev/vdb1

sudo parted /dev/vdb rm 1


sudo dnf install -y jq



✅ 5. (Opcional) Abre la UI de Longhorn en tu navegador
Si usas Traefik o Ingress, verifica si hay un Ingress creado:

bash
Copiar
Editar
kubectl get ingress -n longhorn-system
Si no tienes Ingress, puedes hacer port-forward:

bash
Copiar
Editar
kubectl port-forward -n longhorn-system service/longhorn-frontend 8080:80
Y luego abre tu navegador en:

arduino
Copiar
Editar
http://localhost:8080