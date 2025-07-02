# OKE Zabbix

To deploy Zabbix (with Prometheus flags and exporters), run the following steps in order, monitoring the **Ready**, **Status**, and **Logs** before proceeding to the next step.
**Note**: This deployment does not use persistent volumes (PV) or persistent volume claims (PVC), so all data is **ephemeral** and will be lost if the pod is restarted.

## Requirements

1. Edit **zabbix-ingress.yaml** and replace with your own domain, as shown in the example below:
```bash
# Old
  tls:
    - hosts:
        - yourdomain.domain.com
      secretName: zabbix-tls
  rules:
    - host: yourdomain.domain.com
# New
  tls:
    - hosts:
        - zabbix.example.com
      secretName: zabbix-tls
  rules:
    - host: zabbix.example.com
```

2. Edit **zabbix-configmap.yaml** and replace PHP_TZ and TZ with your Timezone, as shown in the example below:
```bash
# Old
  PHP_TZ: "America/Sao_Paulo"
  TZ: "America/Sao_Paulo"
# New
  PHP_TZ: "America/Santiago"
  TZ: "America/Santiago"
```

## Create Namespace

1. Run the following command to create the namespace:
```bash
kubectl create ns zabbix # Change to another name like "monitoring" if desired
```

## MySQL Deployment

Before starting, you may edit **zabbix-secrets.yaml** to generate your own secrets. You can also skip to **step 3** and use the default values.

1. To change the default MySQL credentials (user/password: zabbix/zabbix) and the Zabbix Web login (Admin/zabbix), run:
```bash
echo -n 'yoursecretlikeaccountorpasswordorotherthing' | base64 -w 0 # "-w 0" avoids line breaks
# This return: eW91cnNlY3JldGxpa2VhY2NvdW50b3JwYXNzd29yZG9yb3RoZXJ0aGluZw==
```

2. Copy and paste the result in the **data** section of **zabbix-secrets.yaml**:
```bash
# Old
MYSQL_PASSWORD: emFiYml4

# New
MYSQL_PASSWORD: eW91cnNlY3JldGxpa2VhY2NvdW50b3JwYXNzd29yZG9yb3RoZXJ0aGluZw==
```

3. Deploy Secrets and ConfigMap using the manifests:
```bash
kubectl apply -f zabbix-secrets.yaml -f zabbix-configmap.yaml

# Check
kubectl get secrets,configmap -n zabbix
```

4. Deploy MySQL using the manifests:
```bash
kubectl apply -f ./mysql

# Wait for Ready=1/1 and Status=Running
kubectl get pods -n zabbix -l app=mysql -w

# Ctrl+C to stop watching, then check logs to verify if it’s waiting for new connections
kubectl logs -n zabbix -l app=mysql
```

## Zabbix Server Deployment

1. Deploy Zabbix Server using the manifests:
```bash
kubectl apply -f ./zabbix

# Wait for Ready=2/2 and Status=Running
kubectl get pods -n zabbix -l app=zabbix -w

# Ctrl+C to stop watching, then check logs to see if it's connected to the database
kubectl logs -n zabbix -l app=zabbix
```

## Zabbix Web Deployment

1. Deploy Zabbix Web using the manifests:
```bash
kubectl apply -f ./web

# Wait for Ready=1/1 and Status=Running
kubectl get pods -n zabbix -l app=web -w

# Ctrl+C to stop watching, check the logs to ensure it started successfully
kubectl logs -n zabbix -l app=web

# Check if the pods are "Running", the certificate is "True", and the other resources exist
kubectl get pods,svc,certificate,secret,configmap,ingress -n zabbix
```

## Exporter Deployment (Optional)

To monitor MySQL, Zabbix Server via Prometheus, follow these steps:

1. Access the MySQL pod:
```bash
kubectl exec -n zabbix $(kubectl get pods -n zabbix -l app=mysql -o jsonpath="{.items[0].metadata.name}") -it -- /bin/bash # or just bash
```

2. Inside the pod, log into MySQL:
```bash
mysql -u root -p
# Password is "zabbix" by default
```

3. Create the exporter user or other that you want:
```bash
CREATE USER 'exporter'@'%' IDENTIFIED BY 'exporter';

GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'exporter'@'%';

FLUSH PRIVILEGES;
```

4. Exit MySQL and optionally test the new user:
```bash
exit # Exit MySQL

# Optional test
mysql -u exporter -p
# Enter "exporter" as password
exit # Exit MySQL again
exit # Exit pod shell
```

5. If you created a new user/password, update the secret before continuing. Otherwise, go to **step 8**:
```bash
echo -n 'yournewuser:yournewpassword@tcp(mysql:3306)/' | base64 -w 0
# This return: eW91cm5ld3VzZXI6eW91cm5ld3Bhc3N3b3JkQHRjcChteXNxbDozMzA2KS8=
```

6. Update **zabbix-secrets.yaml**:
```bash
# Old
DATA_SOURCE_NAME: ZXhwb3J0ZXI6ZXhwb3J0ZXJAdGNwKG15c3FsOjMzMDYpLw==

# New
DATA_SOURCE_NAME: eW91cm5ld3VzZXI6eW91cm5ld3Bhc3N3b3JkQHRjcChteXNxbDozMzA2KS8=
```

7. Re-apply the updated secret:
```bash
kubectl apply -f zabbix-secrets.yaml
```

8. Deploy Exporters using the manifests:
```bash
kubectl apply -f ./exporter

# Wait for Ready=1/1 and Status=Running
kubectl get pods -n zabbix -w

# Press Ctrl+C to stop watching the logs/output.
```