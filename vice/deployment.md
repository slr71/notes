# VICE Deployment

## Service Account

- Need to create a service account for the `app-exposer` service. This has been done in the `k8s-resources` repo for
  TUGraz.

## Cluster Role

- Fortunately, we can reuse the `admin` cluster role for the `app-exposer` service account.

## Cluster Role Binding

- The cluster role needs to be bound to the service account. This has been done in the `k8s-resources` repo for TUGraz.

## Deployments to Modify

- The service account needs to be specified in the `app-exposer` deployment. This has been done in the `k8s-resources`
  repo for TUGraz.

## Deployments to...uh...Deploy

- analyses
- app-exposer
- check-resource-access
- interapps-landing
- timelord
- vice-loading

## Ingresses

- Define the nginx ingress controller. There are some values for the controller defined in the `k8s-resources`
  repository for TUGraz.
