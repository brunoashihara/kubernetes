# OKE Nginx

This is a simple project that deploys Nginx (with Prometheus flags and exporter), along with two ConfigMaps: one for **nginx-conf** and another for the **index.html** file.

## Requirements

Edit **nginx-ingress.yaml** and replace with your own domain, as shown in the example below:
```bash
# Old
  tls:
    - hosts:
        - yourdomain.domain.com
      secretName: nginx-tls
  rules:
    - host: yourdomain.domain.com
# New
  tls:
    - hosts:
        - nginx.example.com
      secretName: nginx-tls
  rules:
    - host: nginx.example.com
```

## Create Namespace

1. Run the following command to create the namespace:
```bash
kubectl create ns nginx # Change to another name like "web" if desired
```

## Nginx Deployment

Before starting, you may edit **nginx-index.yaml** to customize the **index.html** file. You can also skip **step 1** and use the default content.

1. To modify the ConfigMap for **index.html**, open **nginx-index.yaml** and update it as follows:
```bash
# Old
data:
  index.html: |
    <!DOCTYPE html>
    <html lang="pt-BR">
    <body>
      <h1>Hello World!</h1>
    </body>
    </html>
# New
data:
  index.html: |
    <!DOCTYPE html>
    <html lang="pt-BR">
    <body>
      <h1>Goodbye World!</h1>
    </body>
    </html>
```

2. Deploy Nginx:
```bash
kubectl apply -f ./

# Check if the pods are "Running", the certificate is "True", and the other resources exist
kubectl get pod,svc,configmap,certificate,secrets -n nginx
```