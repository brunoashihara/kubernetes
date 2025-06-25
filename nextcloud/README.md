# OKE Nextcloud

To deploy Nextcloud (with Prometheus flags), run the following steps in order, monitoring the **Ready**, **Status**, and **Logs** before proceeding to the next step.
**Note**: This deployment does not use persistent volumes (PV) or persistent volume claims (PVC), so all data is **ephemeral** and will be lost if the pod is restarted. Also, the same password is used for both MariaDB and Nextcloud.

## Requirements

1. Edit **nextcloud-deploy.yaml** and replace with your own domain and the public IP associated with it, as shown in the example below:
```bash
# Old
            - name: NEXTCLOUD_TRUSTED_DOMAINS
              value: |
                yourdomain.domain.com
                0.0.0.0
            - name: NEXTCLOUD_URL
              value: https://yourdomain.domain.com
# New
            - name: NEXTCLOUD_TRUSTED_DOMAINS
              value: |
                nextcloud.domain.com
                10.10.10.10
            - name: NEXTCLOUD_URL
              value: https://nextcloud.domain.com
```

2. Edit **nextcloud-ingress.yaml** and replace the domain with your own, as shown below:
```bash
# Old
  tls:
    - hosts:
        - yourdomain.domain.com
      secretName: nextcloud-tls
  rules:
    - host: yourdomain.domain.com
# New
  tls:
    - hosts:
        - nextcloud.example.com
      secretName: nextcloud-tls
  rules:
    - host: nextcloud.example.com
```

## Create Namespace

1. Run the following command to create the namespace:
```bash
kubectl create ns nextcloud # Change to another name like "file" if desired
```

## MariaDB Deployment

Before starting, you may edit **mariadb-secrets.yaml** to generate your own secrets. You can also skip to **step 3** and use the default values.

1. To change the default MariaDB credentials (user/password: nextcloud/nextcloud) and the nextcloud Web login (admin/nextcloud), run:
```bash
echo -n 'yoursecretlikeaccountorpasswordorotherthing' | base64 -w 0 # "-w 0" avoids line breaks
# This return: eW91cnNlY3JldGxpa2VhY2NvdW50b3JwYXNzd29yZG9yb3RoZXJ0aGluZw==
```

2. Copy and paste the result in the **data** section of **mariadb-secrets.yaml**:
```bash
# Old
MYSQL_PASSWORD: bmV4dGNsb3Vk

# New
MYSQL_PASSWORD: eW91cnNlY3JldGxpa2VhY2NvdW50b3JwYXNzd29yZG9yb3RoZXJ0aGluZw==
```

3. Deploy the secrets:
```bash
kubectl apply -f mariadb-secrets.yaml

# Check
kubectl get secrets -n nextcloud
```

4. Deploy MariaDB and its service, then check the logs:
```bash
kubectl apply -f mariadb-deploy.yaml -f mariadb-svc-yaml

# Wait for Ready=1/1 and Status=Running
kubectl get pods -n nextcloud -l app=mariadb -w

# Ctrl+C to stop watching, then check logs to verify if it’s waiting for new connections
kubectl logs -n nextcloud -l app=mariadb
```

## Redis Deployment

1. Deploy Redis and its service, then check the logs:
```bash
kubectl apply -f redis-deploy.yaml -f redis-svc.yaml

# Wait for Ready=1/1 and Status=Running
kubectl get pods -n nextcloud -l app=redis -w

# Ctrl+C to stop watching, then check logs to ensure Redis started successfully
kubectl logs -n nextcloud -l app=redis
```

## Nextcloud Deployment

1. Deploy Nextcloud, its service, and the ingress with CertManager:
```bash
kubectl apply -f nextcloud-deploy.yaml -f nextcloud-svc.yaml -f nextcloud-ingress.yaml

# Wait for Ready=1/1 and Status=Running
kubectl get pods -n nextcloud -l app=nextcloud -w

# Ctrl+C to stop watching, then check the logs to ensure Nextcloud started successfully
kubectl logs -n nextcloud -l app=nextcloud

# Wait for certificate to become Ready
kubectl get certificate -n nextcloud -w

# Ctrl+C to stop watching, then check if secret was created
kubectl get secrets -n nextcloud
```