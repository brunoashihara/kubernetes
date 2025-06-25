# OKE Vault

In this project, we create a Vault pod that uses internal node storage. By default, Vault creates a Block Volume for each replica to maintain data persistence.

 - Check the configurations if you want to change, for example, the size of the PV and PVC.
 - **Note**: Since we are using internal storage, it's very important to be aware that the data will be tied to a single node.
```bash
  nodeSelector:
    kubernetes.io/hostname: PUTYOURNODENAME
```

## Requirements

Edit **vault-ingress.yaml** and replace with your own domain, as shown in the example below:
```bash
# Old
  tls:
    - hosts:
        - yourdomain.domain.com
      secretName: vault-tls
  rules:
    - host: yourdomain.domain.com
# New
  tls:
    - hosts:
        - vault.example.com
      secretName: vault-tls
  rules:
    - host: vault.example.com
```

## Preparing the Environment

1. Apply the **daemonset-create-vault-dir.yaml** to create local disks on the nodes:
```bash
kubectl apply -f daemonset-create-vault-dir.yaml
```

## Directory Checks and Adjustments

1. Apply **debug-hostpath.yaml** to deploy a busybox container to verify the directory and adjust permissions if needed:
```bash
kubectl apply -f debug-hostpath.yaml
```

2. Use the following to check if the directory was created:
```bash
kubectl exec nomepod -it -- ls -la /mnt/data/vault
```

3. If necessary, apply permissions to the directory:
```bash
kubectl exec nomepod -it -- chmod -R 777 /mnt/data/vault
```

4. For more control, you can set the ownership as follows:
```bash
kubectl exec nomepod -it -- chown -R 100:100 /mnt/data/vault
```

## Creating the Namespace, PV, and PVC

1. Create the namespace:
```bash
kubectl create ns vault
```

2. Deploy the persistent volume (PV):
```bash
kubectl apply -f pv-vault-local.yaml
```

3. Deploy the persistent volume claim (PVC):
```bash
kubectl apply -f pvc-vault-local.yaml
```

4. Monitor the status of the PV and PVC. Wait until the status shows as "Bound". **Press Ctrl+C** to exit:
```bash
kubectl get pv,pvc -n vault -w
```

## Vault Deployment
1. Now deploy Vault using Helm. Edit the hostname in the **values.yaml** file before running:
```bash
helm upgrade --install vault hashicorp/vault -n vault -f values.yaml
```

2. Check the Vault logs. If you see messages like **seal configuration missing, not initialized**, it means Vault is waiting for unseal tokens:
```bash
kubectl logs -n vault vault-0
```

3. Run the following command to get the unseal and root tokens:
```bash
kubectl exec -it vault-0 -n vault -- vault operator init
```

4. Unseal Vault by running the command below three times, using a different unseal key each time:
```bash
kubectl exec -it vault-0 -n vault -- vault operator unseal
```