# Simplifying Configuration

## Demo

```bash
minikube start

kubectl get all -A

kubectl apply -f nginx.yaml

kubectl rollout status deployment/nginx-deployment -n nginx

kubectl get all -n nginx

kubectl delete -f nginx.yaml
```
