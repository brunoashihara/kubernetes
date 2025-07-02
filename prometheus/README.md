# OKE Prometheus

This is a simple project — just run the Helm command below.  

> **Note**:
> This deployment does not use persistent volumes (PV) or persistent volume claims (PVC), so all data is **ephemeral** and will be lost if the pod is restarted.

## Requirements

1. Edit **prometheus-ingress.yaml** and replace with your own domain, as shown in the example below:
```bash
# Old
  tls:
    - hosts:
        - yourdomain.domain.com
      secretName: prometheus-tls
  rules:
    - host: yourdomain.domain.com
# New
  tls:
    - hosts:
        - prometheus.example.com
      secretName: prometheus-tls
  rules:
    - host: prometheus.example.com
```

## Create Namespace

1. Run the following command to create the namespace:
```bash
kubectl create ns prometheus # Change to another name like "monitoring" if desired
```

## Create Secret

1. Create a secret to enforce Basic Authentication in the Prometheus UI:
```bash
htpasswd -c auth prometheusadmin
# Type the password twice — this will create the "auth" file

# Now deploy the secret to be used by Prometheus UI
kubectl create secret generic prometheus-basic-auth --from-file=auth -n prometheus

# Check
kubectl get secret -n prometheus
```

## Prometheus Deployment

1. Run the following Helm commands to install Prometheus:
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack -n prometheus --create-namespace -f values.yaml
```

2. Deploy Ingress and CertManager:
```bash
kubectl apply -f prometheus-ingress.yaml

# Check if the pods are "Running", and the other resources exist
kubectl get pods,svc,certificate,secrets,configmap,ingress -n prometheus
```

## ServiceMonitors Deployment

At this point just run the following command to deploy all the ServiceMonitors:
```bash
kubectl apply -f ./servicemonitor

# Check the ServiceMonitors
kubectl get servicemonitors -n prometheus
```
