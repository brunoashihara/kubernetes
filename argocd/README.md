# OKE ArgoCD

This is a simple project — just run the Helm command below.  

> **Note**:
> This deployment does not use persistent volumes (PV) or persistent volume claims (PVC), so all data is **ephemeral** and will be lost if the pod is restarted.

## Requirements

1. Edit **argocd-ingress.yaml** and replace with your own domain, as shown in the example below:
```bash
# Old
  tls:
    - hosts:
        - yourdomain.domain.com
      secretName: argocd-tls
  rules:
    - host: yourdomain.domain.com
# New
  tls:
    - hosts:
        - argocd.example.com
      secretName: argocd-tls
  rules:
    - host: argocd.example.com
```

2. Edit **values.yaml** and replace with your own github, username and GHP, as shown in the example below:
```bash
# Old
configs:
  cm:
    admin.enabled: "true"
    timeout.reconciliation: "180s"
    theme: "dark"
    repository.credentials: |
      - url: https://github.com/yourgithub
        username: yourusername
        password: yourcreatedghp
# New
configs:
  cm:
    admin.enabled: "true"
    timeout.reconciliation: "180s"
    theme: "dark"
    repository.credentials: |
      - url: https://github.com/examplegithub
        username: example
        password: ghp_restoftheghp
```

## ArgoCD Deployment

1. Run the following Helm commands to install ArgoCD:
```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

helm upgrade --install argocd argo/argo-cd -n argocd --create-namespace --values values.yaml
```

2. Retrieve the initial admin password with the command below:
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

3. Deploy Ingress and CertManager:
```bash
kubectl apply -f argocd-ingress.yaml

# # Check if the pods are "Running", the certificate is "True", and the other resources exist
kubectl get pods,svc,certificate,secret,configmap,ingress -n argocd
```