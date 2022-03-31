# argo-rollouts

This repository documents experiments with ArgoCD rolouts.

NOTE: All commands shown in this readme are assumed to be for a Linux shell. 

## Create an EKS Cluster

1. Create an EKS cluster using your preferred mechanism. For example:
    ```
    eksctl create cluster -f kube-demo.yaml
    ```
1. Test access to the cluster
    ```
    kubectl get svc
    ```

## Deploy ArgoCD to the cluster

1. Create a namespace and deploy ArgoCD
    ```
    kubectl create namespace argocd
    kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
    ```
1. Change ArgoCD service type to load balancer
    ```
    kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
    ```
1. Wait for all ArgoCD pods to be running
    ```
    kubectl get pods -n argocd
    ```
1. Get url to access the ArgoCD console
    ```
    kubectl get service argocd-server -n argocd -o jsonpath="{.status.loadBalancer.ingress[0].hostname}"; echo
    ```
1. Get ArgoCD `admin` user's password
    ```
    kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
    ```
1. Navigate to the url and enter `admin` for username and the password returned by the previous command. Note, load balancer may take some time to be provisioned.

## Install ArgoCD rollouts

Instructions for this section are based on the following resource:
* https://argoproj.github.io/argo-rollouts/installation/#controller-installation

### Controller Installation

1. Create argo-rollouts namespace and install controller
    ```
    kubectl create namespace argo-rollouts
    kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
    ```
1. Check that the argocd-rollouts-xxx pod is running and ready
    ```
    kubectl get pods -n argo-rollouts
    ```

### Kubectl Plugin Installation

1. Install Argo Rollouts Kubectl plugin with curl.
    ```
    curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-linux-amd64
    ```
1. Make the kubectl-argo-rollouts binary executable.
    ```
    chmod +x ./kubectl-argo-rollouts-linux-amd64
    ```
1. Move the binary into your PATH.
    ```
    sudo mv ./kubectl-argo-rollouts-linux-amd64 /usr/local/bin/kubectl-argo-rollouts
    ```
1. Test to ensure the version you installed is up-to-date
    ```
    kubectl argo rollouts version
    ```

## Run the canary deployment example

1. Clone the argocd rollouts-demo repo to your machine
    ```
    git clone https://github.com/argoproj/rollouts-demo.git
    ```
1. Change to the folder where the rollouts-demo repo was cloned
    ```
    cd rollouts-demo
    ```
1. Apply the manifests of one of the examples
    ```
    kubectl kustomize examples/canary | kubectl apply -f -
    ```
1. Expose the example services
    ```
    kubectl patch svc canary-demo -p '{"spec": {"type": "LoadBalancer"}}'
    kubectl patch svc canary-demo-preview -p '{"spec": {"type": "LoadBalancer"}}'
    ```
1. Note the service endpoints for the canary-demo and canary-demo-preview services from the output of this command:
    ```
    kubectl get services
    ```
1. Open a browser window for each of the services - not it may take a while for the service to become available.
    Each window generates load to the canary-demo and canary-demo preview services respectively.
1. Watch the rollout
    ```
    kubectl argo rollouts get rollout canary-demo --watch
    ```
1. In a new command window trigger an update by setting the image of a new color to run
    ```
    kubectl argo rollouts set image canary-demo "*=argoproj/rollouts-demo:yellow"
    ```
1. Wait for the rollout to enter a paused state
1. Notice how some of the traffic in the canary-demo-preview service is switched to yellow. All traffic in the canary-demo service
    remains unaffected.
1. Promote the rollout
    ```
    kubectl argo rollouts promote canary-demo
    ```
1. Note how the rollout is gradually promoted and how all traffic in the canary-demo service is eventually switch to yellow.
1. Alternatively try aborting the rollout
    ```
    kubectl argo rollouts abort canary-demo
    ```

## Cleanup

1. Remove the canary demo
    ```
    kubectl kustomize examples/canary | kubectl delete -f -
    ```
1. Remove argo rollouts
    ```
    kubectl delete namespace argo-rollouts
    ```
1. Remove argocd
    ```
    kubectl delete namespace argocd
    ```
1. Delete the cluster
    ```
    eksctl delete cluster --name kube-demo --region us-west-1
    ```