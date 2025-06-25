# OKE Zabbix

To deploy Zabbix (with Prometheus flags and exporters), run the following steps in order, monitoring the **Ready**, **Status**, and **Logs** before proceeding to the next step.
**Note**: This deployment does not use persistent volumes (PV) or persistent volume claims (PVC), so all data is **ephemeral** and will be lost if the pod is restarted.

## Requirements

Edit **zabbix-ingress.yaml** and replace with your own domain, as shown in the example below:
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

3. Deploy the secrets and config map:
```bash
kubectl apply -f zabbix-secrets.yaml -f zabbix-configmap.yaml

# Check
kubectl get secrets,configmap -n zabbix
```

4. Deploy the MySQL and check the logs:
```bash
kubectl apply -f zabbix-db.yaml

# Wait for Ready=1/1 and Status=Running
kubectl get pods -n zabbix -l app=zabbix-db -w

# Ctrl+C to stop watching, then check logs to verify if it’s waiting for new connections
kubectl logs -n zabbix -l app=zabbix-db
```

## Zabbix Server Deployment

1. Deploy the Zabbix Server (includes Zabbix Agent) using **zabbix-server.yaml**:
```bash
kubectl apply -f zabbix-server.yaml

# Wait for Ready=2/2 and Status=Running
kubectl get pods -n zabbix -l app=zabbix-server -w

# Ctrl+C to stop watching, then check logs to see if it's connected to the database
kubectl logs -n zabbix -l app=zabbix-db
```

## Zabbix Web Deployment

1. Deploy the web interface and ingress with CertManager:
```bash
kubectl apply -f zabbix-web.yaml

# Wait for Ready=1/1 and Status=Running
kubectl get pods -n zabbix -l app=zabbix-web -w

# Ctrl+C to stop watching, check the logs to ensure it started successfully
kubectl logs -n zabbix -l app=zabbix-web

# Now deploy the ingress
kubectl apply -f zabbix-ingress.yaml

# Wait for certificate to become Ready
kubectl get certificate -n zabbix -w

# Ctrl+C to stop watching, then check if secret was created
kubectl get secrets -n zabbix
```

## Zabbix MySQL Exporter Deployment (Optional)

If you have Prometheus and want to monitor MySQL, follow these steps:
1. Access the MySQL pod:
```bash
kubectl exec -n zabbix $(kubectl get pods -n zabbix -l app=zabbix-db -o jsonpath="{.items[0].metadata.name}") -it -- /bin/bash # or just bash
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
echo -n 'yournewuser:yournewpassword@tcp(zabbix-db.zabbix.svc.cluster.local:3306)/' | base64 -w 0
```

6. Update **zabbix-secrets.yaml**:
```bash
# Old
DATA_SOURCE_NAME: ZXhwb3J0ZXI6ZXhwb3J0ZXJAdGNwKHphYmJpeC1kYi56YWJiaXguc3ZjLmNsdXN0ZXIubG9jYWw6MzMwNikv

# New
DATA_SOURCE_NAME: YOURNEEEEWOUTPUT
```

7. Re-apply the updated secret:
```bash
kubectl apply -f zabbix-secrets.yaml
```

8. Deploy the MySQL exporter:
```bash
kubectl apply -f zabbix-db-exporter.yaml

# Wait for Ready=1/1 and Status=Running
kubectl get pods -n zabbix -l app=zabbix-db-exporter -w

# Press Ctrl+C to stop watching the logs/output.
```