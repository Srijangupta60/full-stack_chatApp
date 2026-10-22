# Kubernetes Deployment

This folder contains Kubernetes manifests for deploying the chat application in the `chat-app` namespace. The application consists of a React frontend served by Nginx, a Node.js/Express backend, and MongoDB.

## Manifest overview

| File | Resource | Purpose |
| --- | --- | --- |
| `namespace.yml` | Namespace | Creates the `chat-app` namespace used by the application resources. |
| `secrets.yml` | Secret | Provides the `jwt` value consumed by the backend as `JWT_SECRET`. The value under `data` must be base64-encoded. |
| `mongodb-pv.yml` | PersistentVolume | Defines 1 GiB of local `hostPath` storage at `/data/mongodb`, with a `Retain` reclaim policy. |
| `mongodb-pvc.yml` | PersistentVolumeClaim | Requests 1 GiB of `ReadWriteOnce` storage using the `local` storage class. |
| `mongodb-Deployment.yml` | Deployment | Runs one MongoDB pod and references the persistent volume claim. Check that the container has a matching `volumeMounts` entry for `/data/db` before relying on persistent database storage. |
| `mongodb-Service.yml` | Service | Exposes MongoDB inside the namespace on port `27017`. |
| `backend-Deployment.yml` | Deployment | Runs one backend pod on port `5001`, configures production mode and MongoDB, and reads the JWT secret from `chatapp-secrets`. |
| `backend-Service.yml` | Service | Makes the backend available inside the namespace as `backend:5001`. |
| `frontend-Deployment.yml` | Deployment | Runs one frontend pod, serving the built application on port `80`. |
| `frontend-Service.yml` | Service | Makes the frontend available inside the namespace on port `80`. |
| `ingress.yml` | Ingress | Routes requests for `srijan-chat-bot.com` to the frontend and API traffic to the backend. An Ingress controller must be installed in the cluster for this resource to accept traffic. |

## Prerequisites

- A working Kubernetes cluster and `kubectl` configured to use it.
- Container images referenced by the Deployments must be available to the cluster:
  - `srijandevops/chatapp-backend:latest`
  - `srijandevops/chatapp-frontend:latest`
- The backend environment variables and Secret must contain valid values for the intended environment.
- The cluster must support the `local` storage class and the MongoDB node must have the host path configured. The supplied `hostPath` volume is node-local and is not portable persistent storage for a multi-node production cluster.
- For Ingress access, an Ingress controller and DNS (or a local hosts-file entry) must route `srijan-chat-bot.com` to that controller.

## Apply the manifests

Run these commands from the repository root:

```bash
kubectl apply -f K8S/namespace.yml
kubectl apply -f K8S/
```

Before applying, review `secrets.yml` and the MongoDB connection settings in `backend-Deployment.yml`. Do not store real credentials in source control. Base64 is encoding, not encryption; use an appropriate secret-management workflow for shared or production environments.

## Verify the deployment

```bash
kubectl get pods,services -n chat-app
kubectl get pvc -n chat-app
kubectl get pv
kubectl get ingress -n chat-app
```

If a pod is not ready, inspect its events and logs:

```bash
kubectl describe pod -n chat-app <pod-name>
kubectl logs -n chat-app <pod-name>
```

To test the frontend locally without Ingress, forward a local port to the frontend Service:

```bash
kubectl port-forward -n chat-app service/frontend 8081:80
```

Keep the command running and open `http://localhost:8081` in a browser. Stop port forwarding with `Ctrl+C`.

## Remove the deployment

Remove application resources first, then the namespace:

```bash
kubectl delete -f K8S/
kubectl delete namespace chat-app
```

The PersistentVolume has a `Retain` reclaim policy, so deleting the claim does not automatically remove the underlying data. Review the PV and its host path before cleaning up storage.
