# Argo CD Applications

Kubernetes manifests for deploying applications with [Argo CD](https://argo-cd.readthedocs.io/). This repository currently contains the declarative configuration for the **online-shop** application.

## Repository structure

```text
.
├── README.md
└── declarative_approach/
    └── online_shop/
        ├── online-shop-application.yml
        ├── online-shop-deployment.yml
        └── online-shop-svc.yml
```

## Application components

The `online_shop` directory defines:

- **Argo CD Application** – registers the application with Argo CD and points it to this repository and path.
- **Deployment** – runs five replicas of `amitabhdevops/online_shop:latest` and exposes container port `3000`.
- **Service** – provides an internal `ClusterIP` service named `online-shop-service` on port `3000`.

The deployment requests `128Mi` memory and `100m` CPU per pod, with limits of `256Mi` memory and `500m` CPU.

## Prerequisites

- A Kubernetes cluster with access to the `online-shop` namespace.
- Argo CD installed in the `argocd` namespace.
- The Argo CD instance configured to access this repository.
- Permission to create and manage applications and workloads in the target cluster.

## Deployment with Argo CD

Apply the Argo CD Application manifest to the cluster:

```bash
kubectl apply -f declarative_approach/online_shop/online-shop-application.yml
```

The application is configured to:

- Track the `main` branch.
- Synchronize manifests from `declarative_approach/online_shop`.
- Deploy to the `online-shop` namespace.
- Create the namespace automatically when needed.
- Automatically prune resources removed from Git.
- Automatically self-heal drifted resources.

Check the application status with:

```bash
kubectl get application online-shop -n argocd
kubectl get all -n online-shop
```

To inspect synchronization details:

```bash
argocd app get online-shop
argocd app sync online-shop
```

## Local validation

Before committing manifest changes, validate the YAML and render the Kubernetes resources where possible:

```bash
kubectl apply --dry-run=client -f declarative_approach/online_shop/online-shop-deployment.yml
kubectl apply --dry-run=client -f declarative_approach/online_shop/online-shop-svc.yml
kubectl apply --dry-run=client -f declarative_approach/online_shop/online-shop-application.yml
```

## Updating the application

1. Update the manifests under `declarative_approach/online_shop/`.
2. Commit and push the changes to the tracked branch.
3. Argo CD will detect the changes and synchronize them automatically according to the application policy.
4. Review the sync and workload status after deployment.

When publishing a new container image, prefer a versioned image tag instead of `latest` for reproducible deployments. Update the `image` field in `online-shop-deployment.yml`, commit the change, and allow Argo CD to synchronize the updated manifest.

## Notes

- The service is `ClusterIP`, so it is reachable only from inside the cluster unless an ingress, gateway, port-forward, or another external exposure mechanism is added.
- The Argo CD destination server is configured as `argocd-cluster`; change it if the application should target a different registered cluster.
- The application uses the default Argo CD project. Use a dedicated project and restricted repository/destination permissions for production environments.
