
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
