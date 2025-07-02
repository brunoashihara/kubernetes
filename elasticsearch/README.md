# OKE ElasticSearch

This is a simple project that deploys ElasticSearch (with Prometheus flags and exporter).

> **Note**:
> This deployment does not use persistent volumes (PV) or persistent volume claims (PVC), so all data is **ephemeral** and will be lost if the pod is restarted.

## Requirements

1. Edit **kibana-ingress.yaml** and replace with your own domain, as shown in the example below:
```bash
# Old
  tls:
    - hosts:
        - yourdomain.domain.com
      secretName: kibana-tls
  rules:
    - host: yourdomain.domain.com
# New
  tls:
    - hosts:
        - kibana.example.com
      secretName: kibana-tls
  rules:
    - host: kibana.example.com
```

## Create Namespace

1. Run the following command to create the namespace:
```bash
kubectl create ns elk # Change to another name like "monitoring" if desired
```

## ElasticSearch Deployment

Before starting, you may edit **elasticsearch-secrets.yaml** to generate your own secrets. You can also skip to **step 3** and use the default values.

1. To change the default Elastic credentials (user/password: elastic/elasticpassword) run:
```bash
echo -n 'yoursecretlikeaccountorpasswordorotherthing' | base64 -w 0 # "-w 0" avoids line breaks
# This return: eW91cnNlY3JldGxpa2VhY2NvdW50b3JwYXNzd29yZG9yb3RoZXJ0aGluZw==
```

2. Copy and paste the result in the **data** section of **elasticsearch-secrets.yaml**:
```bash
# Old
BOOTSTRAP_PASSWORD: ZWxhc3RpY3Bhc3N3b3Jk

# New
BOOTSTRAP_PASSWORD: eW91cnNlY3JldGxpa2VhY2NvdW50b3JwYXNzd29yZG9yb3RoZXJ0aGluZw==
```

3. Deploy ElasticSearch using the manifests:
```bash
kubectl apply -f ./elasticsearch

# Wait for Ready=1/1 and Status=Running
kubectl get pods -n elk -l app=elasticsearch -w

# Press Ctrl+C to stop watching, then test the password:
kubectl exec -it -n elk deploy/elasticsearch-master -- curl -u elastic:yourpassword http://localhost:9200
```

## Filebeat Deployment

1. Deploy FileBeat using the manifests:
```bash
kubectl apply -f ./filebeat

# Wait for Ready=1/1 and Status=Running
kubectl logs -n elk -l app.kubernetes.io/name=filebeat -w

# Press Ctrl+C to stop watching, then verify data is reaching ElasticSearch:
kubectl exec -it -n elk deploy/elasticsearch-master -- curl -u elastic:yourpassword http://localhost:9200/_cat/indices?v
```

## Kibana Deployment

Before deploying Kibana, we need to create a service account token:

1. Create token:
```bash
kubectl exec -n elk -it deploy/elasticsearch-master -- curl -u elastic:yourpassword -X POST http://localhost:9200/_security/service/elastic/kibana/credential/token/kibana-token?pretty

# Example response:
E0630 16:45:58.128394   49732 websocket.go:296] Unknown stream id 1, discarding message
                                                                                       {
  "created" : true,
  "token" : {
    "name" : "kibana-token",
    "value" : "yourtoken"
  }

# Encode the token in base64:
echo -n 'yourtoken' | base64 -w 0

#  Result: eW91cnRva2Vu
```

2. Edit **elasticsearch-secrets.yaml** and update it with the new token:
```bash
# Old
data:
  BOOTSTRAP_PASSWORD: Y0A5OHVpSmk0
# New
data:
  BOOTSTRAP_PASSWORD: Y0A5OHVpSmk0
  KIBANA_SERVICE_TOKEN: eW91cnRva2Vu
```

3. Re-apply the updated secret:
```bash
kubectl apply -f ./elasticsearch
```

4. Deploy Kibana using the manifests:
```bash
kubectl apply -f ./kibana

# Wait for Ready=1/1 and Status=Running
kubectl get pods -n elk -l app=kibana -w

# Ctrl+C to stop watching

# Confirm the token
kubectl get secret -n elk elasticsearch-secrets -o jsonpath="{.data.KIBANA_SERVICE_TOKEN}" | base64 -d

kubectl exec -n elk deploy/kibana -- printenv | grep ELASTICSEARCH_SERVICEACCOUNTTOKEN

# If it returns the same token, it's works!
```

## ElasticSearch Exporter Deployment

1. Deploy the ElasticSearch Exporter using the manifests:
```bash
kubectl apply -f ./exporter/elasticsearch-exporter-deploy.yaml -f ./exporter/elasticsearch-exporter-svc.yaml

# Check
kubectl get pods -n elk -l app=elasticsearch-exporter -w

# Ctrl+C to stop watching
```

## Kibana Exporter Deployment

> **Currently under testing**

1. Deploy the Kibana Exporter using the manifests:
```bash
kubectl apply -f kibana-exporter-deploy.yaml -f kibana-exporter-svc.yaml

# # Check if the pods are "Running", the certificate is "True", and the other resources exist
kubectl get pods,svc,certificate,secret,configmap,ingress -n elk
```