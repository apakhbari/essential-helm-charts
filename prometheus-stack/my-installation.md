```
kubectl run test-registry --image=registry.eniac-tech.com/prometheus-stack/busybox:latest --restart=Never -
-command -- sleep 30

kubectl create secret docker-registry internal-registry-secret   --docker-server=registry.eniac-tech.com   --docker-username="kubestg"   --docker-password="rg0vRmPgcs"   -n monitoring

helm install prometheus-stack . -n monitoring --create-namespace -f values.yaml
```