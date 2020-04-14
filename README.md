# Using Linkerd2 with ArgoCD and externally managed TLS

This repo is an example on how we could potentially deploy Linkerd2 using a GitOps / Infrastructure-as-Code model. The following tools are used

- ArgoCD - Manages the applications deployed to our clusters
- Cert Manager - Manages TLS certificates to be used for Linkerd

This example relies on a Linkerd PR <https://github.com/linkerd/linkerd2/pull/4232> and a few additional tweaks to the helm chart from the PR. The tweaks are to allow us to add annotations to the Linkerd api servers and webhooks.

## Repo structure

- `apps` - Describes the applications for ArgoCD to manage in our test cluster
- `manifests` - Contains the helm charts and other manifests to deploy to the cluster. These would usually be stored in a Helm chart repository (ChartMuseum)
- `app-of-apps.yaml` - An ArgoCD `Application` to deploy other applications within the cluster. This is referred to the `app of apps` pattern. <https://argoproj.github.io/argo-cd/operator-manual/cluster-bootstrapping/>

## CI Cluster Setup

For this example, we're going to install everything to the same cluster. In a real environment we would run ArgoCD on a CI cluster with it managing multiple application clusters (clusters for dev, staging, prod, etc)

```bash
kind create cluster
```

### Deploy ArgoCD

From <https://argoproj.github.io/argo-cd/getting_started/#1-install-argo-cd>

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f ./argocd.yaml

brew tap argoproj/tap
brew install argoproj/tap/argocd

kubectl port-forward svc/argocd-server -n argocd 8080:443 &

ADMIN_PASSWORD=$(kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-server -o name | cut -d'/' -f 2)
argocd login localhost:8080 --insecure --username admin --password $ADMIN_PASSWORD
```

### Deploy ArgoCD Managed Applications

#### Prep

I usually disable helm charts from creating their own namespaces and would have Terraform create them. For this example, we'll do it manually

```bash
kubectl create ns cert-manager
kubectl create ns linkerd
```

We will simulate the creation and deployment of the CA public / private keys. I recommend using Azure KeyVault (or similar) and [akv2k8s](https://akv2k8s.io/tutorials/sync/2-certificate/) to generate the CA and apply it to the cluster.

```bash
kubectl -n linkerd create secret tls linkerd-trust-anchor --cert=./example-resources/ca.crt --key=./example-resources/ca.key
```

I prefer to deploy a single instance of ArgoCD to manage multiple clusters. For this example, we'll deploy everything to the same cluster. ArgoCD automatically registers the cluster it's deployed on. We can then apply an ArgoCD `Application` to manage the other applications deployed to the cluster.

**Note** - Update `apps/linkerd.yaml` to set the image to one built from <https://github.com/linkerd/linkerd2/pull/4232>. Failing to do so will cause the proxy-injector, tap and sp-validator pods to crash on startup.

```bash
kubectl apply -n argocd -f ./app-of-apps.yaml
```

Login to the ArgoCD ui via `http://localhost:8080`. Username `admin` password is in the environment variable we set during the ArgoCD deployment, `$ADMIN_PASSWORD`

## Example automated workflow

Using Terraform:

- Create a new Kubernetes cluster
- Create a CA TLS cert.

  To avoid storing the private key in the Terraform state, [akv2k8s](https://akv2k8s.io/tutorials/sync/2-certificate/) could be used to sync a Kubernetes secret with a certificate generated by Azure KeyVault

  - If using Terraform to generate the private key. Save the public and private keys to a Kubernetes secret for cert-manager to use
  - Write the public key to our gitops repository. <https://www.terraform.io/docs/providers/github/r/repository_file.html>

- Against the cluster running ArgoCD
  - Register the new cluster with ArgoCD by creating a secret containing the cluster details
  - Apply an `Application` manifest describing what applications to deploy to the new cluster (simulated via `app-of-apps.yaml`)