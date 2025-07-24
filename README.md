# Document 
## install all dependencies
### Install cert-manager for SSL certificates
```
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.12.0/cert-manager.yaml
```
### Install NGINX ingress controller
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/cloud/deploy.yaml
```
### Wait for services to be ready
```
kubectl wait --for=condition=available --timeout=300s deployment -n cert-manager --all
```
```
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=ingress-nginx -n ingress-nginx --timeout=300s
```
