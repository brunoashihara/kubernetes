# Kubernetes Labs

Here you can find some of the deployments I've been working on in an Always Free OKE (Oracle Kubernetes Engine) environment. I set it up following the [raphaBorges](https://github.com/Rapha-Borges/oke-free) guide, and I'm currently making a few adjustments. I’ll provide more details about these projects later.

> **Note:**  
> This project focuses on using the Always Free OKE tier with 2 nodes (2 vCPUs, 12 GB RAM, and 50 GB each). It uses a Network Load Balancer (NLB) to expose deployments — no Block Volumes, PersistentVolumes (PVs), PersistentVolumeClaims (PVCs), or additional LoadBalancers are created. Except for the ones I describe on **README.md** files  
>  
> Most of the workloads use **ephemeral** storage, meaning that **data will be lost after a restart or node rescheduling**. This approach keeps the environment within the free tier limits.