# OKE Gitea

To deploy Gitea (with Prometheus flags and exporters), run the following steps in order, monitoring the **Ready**, **Status**, and **Logs** before proceeding to the next step.

> **Note**:
> This deployment does not use persistent volumes (PV) or persistent volume claims (PVC), so all data is **ephemeral** and will be lost if the pod is restarted.
> I’m building and pushing the image to my private registry to be used by the Act-Runner. In this project, it uses the internal DNS to access the registry.
> The **GITEA_INSTANCE_URL** uses the internal DNS as well, but you can change it to the external URL if needed.

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

2. Edit **values.yaml** and replace if you have other image that works, as shown in the example below:
```bash
# Old
    actions:
      ENABLE_WEBHOOKS: true
      RUNNER_CAPACITY: 2
      runner:
        image:
          repository: yourdomain.domain.com/act-runner
          tag: latest
# New
    actions:
      ENABLE_WEBHOOKS: true
      RUNNER_CAPACITY: 2
      runner:
        image:
          repository: registry.example.com/act-runner
          tag: latest
```

3. Edit **runner-deploy.yaml** and replace if you have other image that works, as shown in the example below:
```bash
# Old
    spec:
      containers:
        - name: runner
          image: registry.registry.svc.cluster.local:5000/act-runner:latest
# New
    spec:
      containers:
        - name: runner
          image: registry.example.com/act-runner:latest
```

4. Edit **gitea-configmap.yaml** and replace with your own DOMAIN, GITEA_INSTANCE_URL, and any others that you want, as shown in the example below:
```bash
# Old
data:
  GITEA_INSTANCE_URL: "http://gitea-http.gitea.svc.cluster.local:3000"
  GITEA_RUNNER_NAME: "k8s-runner"
  GITEA_RUNNER_LABELS: "k8s,terraform,git,ci,kubectl,helm"
  GITEA_RUNNER_CONFIG: "/data"
# New
data:
  GITEA_INSTANCE_URL: "https://gitea.example.com"
  GITEA_RUNNER_NAME: "k8s-runner"
  GITEA_RUNNER_LABELS: "k8s,terraform,git,ci,kubectl,helm"
  GITEA_RUNNER_CONFIG: "/data"
```

## Create Namespace

1. Run the following command to create the namespace:
```bash
kubectl create ns gitea # Change to another name like "git" if desired
```

## Gitea Deployment

Before starting, you may edit **gitea-secrets.yaml** to generate your own secrets. You can also skip to **step 3** and use the default values.

1. To change the default Gitea credentials (user/password: gitea/gitea), run:
```bash
echo -n 'giteadmin' | base64 -w 0 # "-w 0" avoids line breaks
# This return: Z2l0ZWFkbWlu
```

2. Copy and paste the result in the **data** section of **gitea-secrets.yaml**:
```bash
# Old
data:
  username: Z2l0ZWE=
  password: Z2l0ZWE=
  GITEA_RUNNER_REGISTRATION_TOKEN: Z2VuZXJpY2hhc2h0b3VzZWhlcmVidXR5b3VuZWVkdG90YWtlZnJvbXlvdXJnaXRlYQ==
  GITEA_ADMIN_PASSWORD: Z2l0ZWE=
  GITEA_DB_PASSWORD: Z2l0ZWE=
# New
data:
  username: Z2l0ZWFkbWlu
  password: Z2l0ZWFkbWlu
  GITEA_RUNNER_REGISTRATION_TOKEN: Z2VuZXJpY2hhc2h0b3VzZWhlcmVidXR5b3VuZWVkdG90YWtlZnJvbXlvdXJnaXRlYQ==
  GITEA_ADMIN_PASSWORD: Z2l0ZWFkbWlu
  GITEA_DB_PASSWORD: Z2l0ZWFkbWlu
```

3. Deploy the Secrets and ConfigMap:
```bash
kubectl apply -f gitea-secrets.yaml -f gitea-configmap.yaml

# Check
kubectl get secrets,configmap -n gitea
```

4. Run the following Helm commands to install Gitea:
```bash
helm repo add gitea-charts https://dl.gitea.io/charts/
helm repo update

helm upgrade --install gitea gitea-charts/gitea -n gitea -f values.yaml --create-namespace
```

5. Deploy Ingress and Cert-Manager:
```bash
kubectl apply -f gitea-ingress.yaml

# # Check if the pods are "Running", the certificate is "True", and the other resources exist
kubectl get pods,svc,certificate,secret,configmap,ingress -n gitea
```

## Act-Runner Deployment

To use Gitea Actions, you must deploy Act-Runner. I used my own image built from my [Registry Deploy]{https://github.com/brunoashihara/kubernetes/tree/main/registry}

1. Connect to the [docker-builder]{https://github.com/brunoashihara/kubernetes/tree/main/docker-builder} or if you already have Docker installed and accessible to the registry, feel free to use your local environment instead.:
```bash
kubectl exec -n docker $(kubectl get pods -n docker -l app=docker-builder -o jsonpath="{.items[0].metadata.name}") -it -- sh
```

2. Install required packages and build the image:
```bash
# Install packages
apk update && apk add curl bash git unzip python3 py3-pip jq make docker

# Create a directory for the Gitea Act-Runner
mkdir gitea && cd gitea

# Create the Dockerfile and entrypoint.sh based on your build-gitea-runner file from the docker-builder directory
vi Dockerfile

vi entrypoint.sh

# Before building and pushing, we need to create a daemon.json to force the use of HTTP
mkdir -p /etc/docker

cat <<EOF | tee /etc/docker/daemon.json
{
  "insecure-registries": ["registry.registry.svc.cluster.local:5000"]
}
EOF

# Kill Docker and restart it
pkill dockerd && dockerd &

# If it doesn't start, you can run pkill again and start it like this:
pkill dockerd
dockerd > /var/log/dockerd.log 2>&1 & 

# Now you can build and push to the private registry
docker build --no-cache -t registry.registry.svc.cluster.local:5000/act-runner:latest .

docker push registry.registry.svc.cluster.local:5000/act-runner:latest
```

3. Exit the pod:
```bash
exit
```

4. Go to the Gitea UI and generate a token to use with Act-Runner:
```bash
echo -n 'exampleoftoken' | base64 -w 0
# This return: ZXhhbXBsZW9mdG9rZW4=
```

5. Update **gitea-secrets.yaml** by adding the encoded token to the data section:
```bash
# Old
apiVersion: v1
kind: Secret
metadata:
  name: gitea-secrets
  namespace: gitea
type: Opaque
data:
  username: Z2l0ZWFfYWRtaW4=
  password: Z2l0ZWE=
  GITEA_ADMIN_PASSWORD: Z2l0ZWE=
  GITEA_DB_PASSWORD: Z2l0ZWE=
# New
apiVersion: v1
kind: Secret
metadata:
  name: gitea-secrets
  namespace: gitea
type: Opaque
data:
  username: Z2l0ZWFfYWRtaW4=
  password: Z2l0ZWE=
  GITEA_ADMIN_PASSWORD: Z2l0ZWE=
  GITEA_DB_PASSWORD: Z2l0ZWE=
  GITEA_RUNNER_REGISTRATION_TOKEN: ZXhhbXBsZW9mdG9rZW4=
```

6. Re-apply the updated secret:
```bash
kubectl apply -f gitea-secrets.yaml
```

7. Deploy Act-Runner using the manifests:
```bash
kubectl apply -f runner-deploy.yaml

# Wait for Ready=1/1 and Status=Running, you can press Ctrl+C to exit watch mode.
kubectl get pods -n gitea -l app=gitea-act-runner -w

# Press Ctrl+C to stop watching the logs/output.
```