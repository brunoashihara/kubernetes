# OKE Grafana

This is a simple project that deploys Grafana (with Prometheus flags).

> **Note**:
> This deployment does not use persistent volumes (PV) or persistent volume claims (PVC), so all data is **ephemeral** and will be lost if the pod is restarted.

## Requirements

1. Edit **grafana-ingress.yaml** and replace with your own domain, as shown in the example below:
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

2. Edit **values.yaml** and replace with your timezone with you want, as shown in the example below:
```bash
# Old
  env:
    - name: TZ
      value: "America/Sao_Paulo"
# New
  env:
    - name: TZ
      value: "America/Santiago"
```

## Create Namespace

1. Run the following command to create the namespace:
```bash
kubectl create ns grafana # Change to another name like "observability" if desired
```

## Grafana Deployment

Before starting, you may edit **grafana-secrets.yaml** to generate your own secrets. You can also skip to **step 3** and use the default values.

1. To change the default Grafana credentials (user/password: admin/admin), run:
```bash
echo -n 'grafanapass' | base64 -w 0 # "-w 0" avoids line breaks
# This return: Z3JhZmFuYXBhc3M=
```

2. Copy and paste the result in the **data** section of **grafana-secrets.yaml**:
```bash
# Old
data:
  admin-user: YWRtaW4=
  admin-password: YWRtaW4=
# New
data:
  admin-user: YWRtaW4=
  admin-password: Z3JhZmFuYXBhc3M=
```

3. Deploy the Secrets:
```bash
kubectl apply -f grafana-secrets.yaml

# Check
kubectl get secrets -n grafana
```

4. Run the following Helm commands to install Grafana:
```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm upgrade --install grafana grafana/loki-stack -n grafana -f values.yaml --create-namespace
```

5. Deploy Ingress and Cert-Manager:
```bash
kubectl apply -f grafana-ingress.yaml
```

6. Deploy Promtail Service:
```bash
kubectl apply -f promtail-svc.yaml

# # Check if the pods are "Running", the certificate is "True", and the other resources exist
kubectl get pods,svc,certificate,secret,configmap,ingress -n grafana
```