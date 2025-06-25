# OKE Registry

This is a simple project that deploys Registry (with Prometheus flags).
**Note**: This deployment does not use persistent volumes (PV) or persistent volume claims (PVC), so all data is **ephemeral** and will be lost if the pod is restarted.

## Requirements

Edit **registry-ingress.yaml** and replace with your own domain, as shown in the example below:
```bash
# Old
  tls:
    - hosts:
        - yourdomain.domain.com
      secretName: registry-tls
  rules:
    - host: yourdomain.domain.com
# New
  tls:
    - hosts:
        - registry.example.com
      secretName: registry-tls
  rules:
    - host: registry.example.com
```

## Create Namespace

1. Run the following command to create the namespace:
```bash
kubectl create ns registry # Change to another name like "cicd" if desired
```

## Registry Deployment

There is nothing much to do here, just deploy the **registry-deploy.yaml**, **registry-svc.yaml**, **registry-configmap.yaml** and **registry-ingress.yaml**:

1. Deploy Registry and ingress with CertManager:
```bash
kubectl apply -f ./

# Check if the pods are "Running", the certificate is "True", and the other resources exist
kubectl get pod,svc,certificate,secrets,configmap -n registry
```