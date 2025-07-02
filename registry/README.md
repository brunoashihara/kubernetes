# OKE Registry

This is a simple project that deploys Registry (with Prometheus flags and exporter).

> **Note**:
> This deployment does not use persistent volumes (PV) or persistent volume claims (PVC), so all data is **ephemeral** and will be lost if the pod is restarted.
> I’m building and pushing the image to my private registry to be used by the exporter. In this project, it uses the internal DNS to access the registry.

## Requirements

1. Edit **registry-ingress.yaml** and replace with your own domain, as shown in the example below:
```bash
# Old
  tls:
    - hosts:
        - yourdomain.domain.com
      secretName: registry-tls
  rules:
    - host: yourdomain.domain.com
# New
  tls:
    - hosts:
        - registry.example.com
      secretName: registry-tls
  rules:
    - host: registry.example.com
```

2. Edit **registry-exporter.deploy.yaml** and replace with your own domain, as shown in the example below:
```bash
# Old
    spec:
      containers:
        - name: registry-exporter
          image: yourdomain.domain.com/registry-exporter:latest
# New
    spec:
      containers:
        - name: registry-exporter
          image: registry.example.com/registry-exporter:latest
```

## Create Namespace

1. Run the following command to create the namespace:
```bash
kubectl create ns registry # Change to another name like "cicd" if desired
```

## Registry Deployment

There is nothing much to do here, just deploy the **registry-deploy.yaml**, **registry-svc.yaml**, **registry-configmap.yaml** , **registry-ingress.yaml**, :

1. Deploy Registry using the manifests:
```bash
kubectl apply -f ./registry

# Check if the pods are "Running", the certificate is "True", and the other resources exist
kubectl get pods,svc,certificate,secret,configmap,ingress -n registry
```

## Exporter Creation

Let's create the exporter image and in the next section we can deploy everything at once. I used my own image built from my [Registry Deploy]{https://github.com/brunoashihara/kubernetes/tree/main/registry}

1. Connect to the [docker-builder]{https://github.com/brunoashihara/kubernetes/tree/main/docker-builder} or if you already have Docker installed and accessible to the registry, feel free to use your local environment instead.:
```bash
kubectl exec -n docker $(kubectl get pods -n docker -l app=docker-builder -o jsonpath="{.items[0].metadata.name}") -it -- sh
```

2. Install required packages and build the image:
```bash
# Install packages
apk update && apk add curl bash git unzip python3 py3-pip jq make docker

# Create a directory for the Gitea Act-Runner
mkdir registry && cd registry

# Create the Dockerfile and registry_exporter.sh based on your build-registry-exporter file from the docker-builder directory
vi Dockerfile

vi registry_exporter.sh

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
docker build --no-cache -t registry.registry.svc.cluster.local:5000/registry-exporter:latest .

docker push registry.registry.svc.cluster.local:5000/registry-exporter:latest
```

3. Exit the pod:
```bash
exit
```

4. Deploy Exporter using the manifests:
```bash
kubectl apply -f ./exporter

# Wait for Ready=1/1 and Status=Running
kubectl get pods -n registry -l app=registry-exporter -w

# Ctrl+C to stop watching
```