# OKE Docker-Builder

This is a simple project that deploys a Docker builder used to create images and push them to the [Registry](https://github.com/brunoashihara/kubernetes/tree/main/registry).

> **Note**:
> This deployment does not use persistent volumes (PV) or persistent volume claims (PVC), so all data is **ephemeral** and will be lost if the pod is restarted.
> If you already have access to a Kubernetes cluster with Docker installed, you can skip this deployment.

## Docker-builder Deployment

1. Just run the following command to install:
```bash
kubectl apply -f docker-builder.yaml

# Check
kubectl get pods -n docker -l app=docker-builder
```

2. Access the Docker builder pod:
```bash
kubectl exec -n docker $(kubectl get pods -n docker -l app=docker-builder -o jsonpath="{.items[0].metadata.name}") -it -- sh
```

3. Install the required packages and configure daemon.json to allow access to the internal registry:
```bash
# Install packages
apk update && apk add curl bash git unzip python3 py3-pip jq make docker


# Create daemon.json
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
```