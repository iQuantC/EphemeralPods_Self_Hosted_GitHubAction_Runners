# Self-Hosted GitHub Actions Runners w/ Ephemeral Kubernetes Pods
In this project, we present a complete and practical DevOps project that shows how to run self-hosted GitHub Actions runners as Kubernetes pods that spin up on demand, execute a CI job, then terminate. It uses Actions Runner Controller (ARC) (the de-facto operator for this use-case).


## Prerequisites
1. GitHub Account & Repository
2. Actions Runner Controller (ARC)
3. Kubernetes cluster (Minikube)
4. Kubectl
5. Helm
6. Cert Manager (ARC uses it for admission webhooks)
7. Runner Scale Set (The component that creates ephemeral pods on-demand)


## Create Kubernetes Cluster
```sh
minikube start --driver=docker
```

```sh
kubectl get nodes
```


## Install Cert Manager (ARC recommends it)
```sh
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.18.2/cert-manager.yaml
```

```sh
kubectl get pods --namespace cert-manager
```


## Install the ARC Operator
This sets up the controller manager and listener components that let ARC orchestrate runner pods.

```sh
NAMESPACE="arc-systems"
helm install arc \
  --namespace "${NAMESPACE}" \
  --create-namespace \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller
```

### Verify the ARC Pods
```sh
kubectl get pods -n "${NAMESPACE}"
helm list -A
```


## Create a GitHub Personal Access Token
The GitHub PAT should have at least "repo", "workflow", "admin:org", and "admin:repo_hook" scopes.


```sh
RUNNER_NS="arc-runners"
GITHUB_PAT="ghp_4WpDHUEq7s11xxx"
```


## Create a Kubernetes Secret with the GitHub PAT

```sh
kubectl create ns ${RUNNER_NS}
```
```sh
kubectl create secret generic arc-github-config \
  --namespace ${RUNNER_NS} \
  --from-literal=github_token="${GITHUB_PAT}"
```


## Install the Runner Scale Set (this scales the pods on demand)

```sh
INSTALLATION_NAME="arc-runner-set"   # this becomes the value to use in `runs-on` in workflows
NAMESPACE="${RUNNER_NS}"
GITHUB_CONFIG_URL="https://github.com/iQuantC/EphemeralPods_Self_Hosted_GitHubAction_Runners"
```

```sh
helm install "${INSTALLATION_NAME}" \
  --namespace "${NAMESPACE}" \
  --create-namespace \
  --set githubConfigUrl="${GITHUB_CONFIG_URL}" \
  --set githubConfigSecret.github_token="${GITHUB_PAT}" \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set
```


## Verify Controller & Scale Set Are Running
```sh
helm list -A
kubectl get pods -n arc-systems
```

You must see the 
1. CONTROLLER POD - which manages the runner lifecycle, and 
2. LISTENER POD   - which establishes connection to GitHub, watches for jobs in your repo, and requests ephemeral runner pods when jobs arrive.
3. These are "always-on" components.


```sh
kubectl get pods -n arc-runners
```
1. This may show no running pods because no job is running.
2. With scale set runners, ARC does not keep idle runners sitting around.
3. Runner pods in arc-runners namespace are only created when GitHub Actions Job is queued that specifies ***runs-on: arc-runner-set***
4. After the job finishes, ARC automatically terminates and deletes the pod.


## Create a Sample CI Workflow with Your Repo
Create a .github/workflows/arc-demo.yml in the root of your repo and add the ff contents: 

```sh
# .github/workflows/arc-demo.yml

name: ARC Demo (on-demand runners)

on:
  workflow_dispatch:
  push:
    branches: [ main ]

jobs:
  build:
    runs-on: arc-runner-set           # this must match the helm INSTALLATION_NAME value 
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Show runner info
        run: |
          echo "This job runs on a self-hosted runner managed by ARC"
          hostname
          cat /etc/os-release || true

      - name: Run a small build/test
        run: |
          python3 --version || true
          echo "Done."
```

By pushing changes or saving this file in GitHub the GitHub Actions Workflow will be triggered and will start running the job.
Otherwise, use the Actions tab -> Run workflow. 


### Verify the ephemeral pods as Workflow Runs
In a dedicated terminal, run the command below and monitor it as the job is executed by Actions
```sh
kubectl get pods -n arc-runners --watch
```
You should see a pod appear for the job, run, then exit and be removed when the job finishes (ephemeral behavior). 


Once job is done, run the command below in a dedicated terminal
```sh
kubectl get pods -n arc-runners
```
No pods should be running at this time (after Actions completes build). 

***LIKE, COMMENT, and SUBSCRIBE TO iQuant on YOUTUBE!!!***


## Clean UP

```sh
helm uninstall arc -n arc-systems
helm uninstall arc-runner-set -n arc-runners
kubectl delete ns arc-systems arc-runners cert-manager
```

```sh
minikube stop
minikube delete --all
```
