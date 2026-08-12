# The Project
This is a simulation of how a microservices e-commerce application is deployed and operated on AWS. Six independently deployed services run on Amazon Elastic Kubernetes Service (EKS), with each service owning its own RDS PostgreSQL database for strict data isolation. The wider infrastructure leans on AWS native tooling throughout ECR for container image storage, S3 for static frontend hosting and product assets, Secrets Manager for credential management, and IAM with IRSA for fine grained, pod level access control, no long lived credentials anywhere in the system. This project is maintained across four repositories ,

1. [Platform Repository](https://github.com/pdgayan/ecom-platform-aws)
2. [Backend Services Repository](https://github.com/pdgayan/ecom-services)
3. [GitOps Repository](https://github.com/pdgayan/ecom-gitops)
4. [Web Repository](https://github.com/pdgayan/ecom-web)


# ecom-gitops

This is the **single source of truth** for what runs on the EKS cluster. It holds every Kubernetes manifest and the ArgoCD Application definitions that wire them to the cluster. The backend CI pipeline writes to this repository — updating image tags in deployment manifests — and ArgoCD continuously reconciles the cluster state to match what is declared here. No one deploys directly to the cluster; all changes flow through a commit to this repository.

---

## GitOps Flow

```
  ecom-backend CI          ecom-gitops (this repo)             EKS Cluster
       │                           │                               │
       │  git push                 │                               │
       │  (new image SHA) ────────►│                               │
       │                           │                               │
       │                    updates deployment.yml                 │
       │                    image: <ecr>/<svc>:<sha>              │
       │                           │                               │
       │                           │◄──── ArgoCD polls main ───────│
       │                           │      every 3 min              │
       │                           │                               │
       │                           │─── diff detected ────────────►│
       │                           │                               │
       │                           │      kubectl apply            │
       │                           │      rolling update           │
       │                           │                               │
       │                           │◄────── selfHeal: true ────────│
       │                           │   (reverts manual changes)    │
       │                           │                               │
```

---

## ArgoCD Application Wiring

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     ArgoCD Applications                                 │
│                                                                         │
│  argocd/applications/auth-service-app.yml                               │
│   ├── source:  ecom-gitops / auth-service/                              │
│   └── dest:    EKS cluster, namespace: ecom                             │
│                                                                         │
│  argocd/applications/catalog-service-app.yml                            │
│   ├── source:  ecom-gitops / catalog-service/                           │
│   └── dest:    EKS cluster, namespace: ecom                             │
│                                                                         │
│  argocd/applications/cart-service-app.yml       ──┐                    │
│  argocd/applications/order-service-app.yml        │  same pattern      │
│  argocd/applications/payment-service-app.yml      │  per service       │
│  argocd/applications/notification-service-app.yml ──┘                  │
│                                                                         │
│  Each Application: prune: true, selfHeal: true                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Kubernetes Manifest Structure (per service)

```
  <service>/
  ├── deployment.yml       ◄── image tag updated by CI on every push
  │     spec:
  │       containers:
  │         image: <ecr>/<service>:<commit-sha>   ← this line changes
  │
  ├── service.yml          ◄── ClusterIP (internal only, fronted by Ingress)
  │
  ├── migration_job.yml    ◄── Kubernetes Job: runs DB migrations before pod starts
  │
  └── <svc>-sa.yml         ◄── ServiceAccount with IRSA annotation
          eks.amazonaws.com/role-arn: arn:aws:iam::<acct>:role/<svc>-role
```

---

## Ingress Routing

```
                     ┌──────────────────────────────────────────────┐
                     │          AWS ALB (Ingress Controller)        │
                     │                                              │
                     │  /api/auth/*      ──────► auth-service:3001  │
                     │  /api/products/*  ──────► catalog-svc:3002   │
                     │  /api/cart/*      ──────► cart-svc:3003      │
                     │  /api/orders/*    ──────► order-svc:3004     │
                     │  /api/payment/*   ──────► payment-svc:3005   │
                     └──────────────────────────────────────────────┘
```

---

## Deployment Lifecycle

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │  Sequence for every service deploy                                   │
  │                                                                      │
  │  1. ArgoCD detects new image SHA in deployment.yml                  │
  │                      │                                               │
  │  2. ArgoCD applies migration_job.yml first (init container pattern) │
  │                      │                                               │
  │  3. Migration Job runs:  node migrate.js  ──► RDS (SQL migrations)  │
  │                      │                                               │
  │  4. Job completes ──► Deployment rolls out new pods (rolling update) │
  │                      │                                               │
  │  5. Old pods terminate after health checks pass                      │
  │                      │                                               │
  │  6. selfHeal:true ensures drift is auto-corrected                   │
  └──────────────────────────────────────────────────────────────────────┘
```

---

## Repository Structure

```
ecom-gitops/
│
├── argocd/
│   ├── argocd-ingress.yml          # Exposes the ArgoCD UI via Ingress
│   └── applications/
│       ├── auth-service-app.yml    # ArgoCD App — watches auth-service/ in this repo
│       ├── catalog-service-app.yml
│       ├── cart-service-app.yml
│       ├── order-service-app.yml
│       ├── payment-service-app.yml
│       └── notification-service-app.yml
│
├── auth-service/
│   ├── deployment.yml              # Deployment manifest (image tag updated by CI)
│   ├── service.yml                 # ClusterIP service
│   ├── migration_job.yml           # Kubernetes Job: run DB migrations on deploy
│   └── auth-service-sa.yml         # ServiceAccount with IRSA annotation
│
├── catalog-service/                # Same structure: deployment, service, migration, SA
├── cart-service/
├── order-service/
├── payment-service/
└── notification-service/
```

---

**How a deployment flows through this repo:**
1. Backend CI pushes a new commit SHA image tag → `yq` updates `<service>/deployment.yml`
2. CI commits and pushes to `main` in this repository
3. ArgoCD detects the diff against the live cluster state (polling `main`)
4. ArgoCD applies the updated manifest — cluster rolls out the new pod version
5. `selfHeal: true` ensures any manual cluster changes are reverted back to this repo's state