# OKE Gitea

To deploy Gitea (with Prometheus flags and exporters), run the following steps in order, monitoring the **Ready**, **Status**, and **Logs** before proceeding to the next step.
**Note**: This deployment uses **emptydir**, so all data is **ephemeral** and will be lost if the pod is restarted.

## Requirements

1. Edit **gitea-ingress.yaml** and replace with your own domain, as shown in the example below:
```bash
# Old
  tls:
    - hosts:
        - yourdomain.domain.com
      secretName: gitea-tls
  rules:
    - host: yourdomain.domain.com
# New
  tls:
    - hosts:
        - gitea.example.com
      secretName: gitea-tls
  rules:
    - host: gitea.example.com
```

2. Edit **values.yaml** and replace with your own username, password, email and domain, as shown in the example below:
```bash
# Old
gitea:
  admin:
    username: gitea_admin
    password: "gitea"
    email: "youremail@email.com"
  metrics:
    enabled: true
    token: ""
    serviceMonitor:
      enabled: false
  config:
    server:
      DOMAIN: yourdoamin.domain.com
      SSH_DOMAIN: yourdoamin.domain.com
      ROOT_URL: https://yourdoamin.domain.com
      HTTP_PORT: 3000
      SSH_PORT: 22
      SSH_LISTEN_PORT: 22
    database:
      DB_TYPE: sqlite3
      HOST: localhost
      NAME: gitea
      USER: gitea
      PASSWD: "gitea"
# New
gitea:
  admin:
    username: gitea_admin
    password: "yourpassword"
    email: "gitea@email.com"
  metrics:
    enabled: true
    token: ""
    serviceMonitor:
      enabled: false
  config:
    server:
      DOMAIN: gitea.domain.com
      SSH_DOMAIN: gitea.domain.com
      ROOT_URL: https://gitea.domain.com
      HTTP_PORT: 3000
      SSH_PORT: 22
      SSH_LISTEN_PORT: 22
    database:
      DB_TYPE: sqlite3
      HOST: localhost
      NAME: gitea
      USER: gitea
      PASSWD: "yourpassword"
```

## Gitea Deployment
1. Deploy Gitea using Helm:
```bash
helm upgrade --install gitea gitea-charts/gitea -n gitea -f values.yaml --create-namespace

# # Check if the pods are "Running", the certificate is "True", and the other resources exist
kubectl get pods,svc,certificate,secret -n gitea
```

## Act-Runner Deployment

Work in progress...