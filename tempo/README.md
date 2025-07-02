# OKE Tempo

This is a simple project that deploys Tempo.

> **Note**:
> This deployment does not use persistent volumes (PV) or persistent volume claims (PVC), so all data is **ephemeral** and will be lost if the pod is restarted.

## Tempo Deployment

1. Run the following Helm commands to install Tempo:
```bash
helm upgrade --install tempo grafana/tempo -n tempo -f values.yaml --create-namespace

# Check if the pods are "Running", and the other resources exist
kubectl get pods,svc,secret,configmap -n tempo
```