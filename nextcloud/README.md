# OKE Nextcloud

To deploy Nextcloud (with Prometheus flags), run the following steps in order, monitoring the **Ready**, **Status**, and **Logs** before proceeding to the next step.

> **Note**:
> This deployment does not use persistent volumes (PV) or persistent volume claims (PVC), so all data is **ephemeral** and will be lost if the pod is restarted. Also, the same password is used for both MariaDB and Nextcloud.

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
MARIADB_PASSWORD: bmV4dGNsb3Vk

# New
MARIADB_PASSWORD: eW91cnNlY3JldGxpa2VhY2NvdW50b3JwYXNzd29yZG9yb3RoZXJ0aGluZw==
```

3. Deploy MariaDB using the manifests:
```bash
kubectl apply -f ./mariadb

# Wait for Ready=1/1 and Status=Running
kubectl get pods -n nextcloud -l app=mariadb -w

# Ctrl+C to stop watching, then check logs to verify if it’s waiting for new connections
kubectl logs -n nextcloud -l app=mariadb
```

## Redis Deployment

1. Deploy Redis using the manifests:
```bash
kubectl apply -f ./redis

# Wait for Ready=1/1 and Status=Running
kubectl get pods -n nextcloud -l app=redis -w

# Ctrl+C to stop watching, then check logs to ensure Redis started successfully
kubectl logs -n nextcloud -l app=redis
```

## Nextcloud Deployment

1. Deploy Nextcloud using the manifests:
```bash
kubectl apply -f ./nextcloud

# Check if the pods are "Running", the certificate is "True", and the other resources exist
kubectl get pods,svc,certificate,secrets,ingress -n nextcloud
```

## Exporter Deployment

To monitor MariaDB, redis and nextcloud via Prometheus, follow these steps:

1. Access the MariaDB pod:
```bash
kubectl exec -n nextcloud $(kubectl get pods -n nextcloud -l app=mariadb -o jsonpath="{.items[0].metadata.name}") -it -- /bin/bash # or just bash
```

2. Inside the pod, log into Mariadb:
```bash
mariadb -u root -p
# Password is "nextcloud" by default
```

3. Create the exporter user or other that you want:
```bash
CREATE USER 'exporter'@'%' IDENTIFIED BY 'exporter';

GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'exporter'@'%';

FLUSH PRIVILEGES;
```

4. Exit MariaDB and optionally test the new user:
```bash
exit # Exit MariaDB

# Optional test
mariadb -u exporter -p
# Enter "exporter" as password
exit # Exit MariaDB again
exit # Exit pod shell
```

5. If you created a new user/password, update the secret before continuing. Otherwise, go to **step 8**:
```bash
echo -n 'yournewuser:yournewpassword@tcp(mariadb:3306)/' | base64 -w 0
# This return: eW91cm5ld3VzZXI6eW91cm5ld3Bhc3N3b3JkQHRjcChtYXJpYWRiOjMzMDYpLw==
```

6. Update **mariadb-secrets.yaml**:
```bash
# Old
DATA_SOURCE_NAME: ZXhwb3J0ZXI6ZXhwb3J0ZXJAKG1hcmlhZGI6MzMwNikv

# New
DATA_SOURCE_NAME: eW91cm5ld3VzZXI6eW91cm5ld3Bhc3N3b3JkQHRjcChtYXJpYWRiOjMzMDYpLw==
```

7. Re-apply the updated secret:
```bash
kubectl apply -f mariadb-secrets.yaml
```

8. Deploy the Exporters using the manifests:
```bash
kubectl apply -f ./exporter

# Wait for Ready=1/1 and Status=Running
kubectl get pods -n nextcloud -w

# Press Ctrl+C to stop watching the logs/output.
```