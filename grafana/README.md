# OKE Grafana

This is a simple project that deploys Grafana (with Prometheus flags).
**Note**: This deployment does not use persistent volumes (PV) or persistent volume claims (PVC), so all data is **ephemeral** and will be lost if the pod is restarted.

## Requirements

Edit **grafana-ingress.yaml** and replace with your own domain, as shown in the example below:
```bash
# Old
  tls:
    - hosts:
        - yourdomain.domain.com
      secretName: grafana-tls
  rules:
    - host: yourdomain.domain.com
# New
  tls:
    - hosts:
        - grafana.example.com
      secretName: grafana-tls
  rules:
    - host: grafana.example.com
```

## Create Namespace

1. Run the following command to create the namespace:
```bash
kubectl create ns grafana # Change to another name like "monitoring" if desired
```

## Grafana Deployment

There is nothing much to do here, just deploy the **grafana.yaml** and **grafana-ingress.yaml**:

1. Deploy Grafana and ingress with CertManager:
```bash
kubectl apply -f ./

# Check if the pods are "Running", the certificate is "True", and the other resources exist
kubectl get pod,svc,certificate,secrets -n grafana
```