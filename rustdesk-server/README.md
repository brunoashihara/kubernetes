# OKE RustDesk

This is a simple project — just run the Helm command below.  
**Note**: This is a test deployment. The **ID Server** is not working; only the **Relay Server** should be configured on the client.

## RustDesk Deployment

1. Run the helm command to install
```bash
helm upgrade --install rustdesk-server ./rustdesk-server -n rustdesk --create-namespace
```