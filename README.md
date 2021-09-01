# Setting up a Kubernetes Cluster for the lab
Before installing LinkerD you need a Kubernetes Cluster. 
1. Create a network that will be used by our application
    ```
    gcloud compute networks create default \
    --subnet-mode=auto \
    --bgp-routing-mode=regional \
    --mtu=1460
    ```
2. Create the GKE Kubernetes cluster
    ```
    gcloud container clusters create linkerd-observability \
    --region us-central1 \
    --enable-ip-alias \
    --enable-network-policy \
    --num-nodes 1 \
    --machine-type "e2-standard-2" \
    --release-channel stable
    ```
3. Authenticate to the GKE cluster
    ```
    gcloud container clusters get-credentials linkerd-observability --region us-central1
    ```

# Getting Started
To install Linkerd on your environment, follow the steps in https://linkerd.io/2.10/getting-started/# or the instructions below: 

1. Install Linkerd
    ```
    curl -sL run.linkerd.io/install | sh
    ```
2. Follow the instructions in the terminal to add the linkerd CLI to your path - it should look something like
    ```
    export PATH=$PATH:/home/sohnya/.linkerd2/bin
    ```
3. Check that the correct client version is installed with
    ```
    linkerd version
    ```
4. Validate that Linkerd can be installed in the Kubernetes cluster
    ```
    linkerd check --pre
    ```
5. Install the control plane into the `linkerd` namespace
    ```
    linkerd install | kubectl apply -f - 
    ```
6. Validate that everything worked
    ```
    linkerd check
    ```

# Word on PodSecurityPolicies
From Kubernetes documentation

`Pod security policy control`: implemented as an optional admission controller. PodSecurityPolicies are enforced by enabling the admission controller, but doing so without authorizing any policies will prevent any pods from being created in the cluster"

How do I verify if the admission controller is enabled? 

`Admission Controller`: An admission controller is a piece of code that intercepts requests to the Kubernetes API server prior to persistence of the object, but after the request is authenticated and authorized. 

Part of the pre-check on my GKE cluster failed with the following messages:

- `‼ has NET_ADMIN capability`: found 1 PodSecurityPolicies, but none provide NET_ADMIN, proxy injection will fail if the PSP admission controller is running
see https://linkerd.io/2.10/checks/#pre-k8s-cluster-net-admin for hints
   
- `‼ has NET_RAW capability`: found 1 PodSecurityPolicies, but none provide NET_RAW, proxy injection will fail if the PSP admission controller is running
see https://linkerd.io/2.10/checks/#pre-k8s-cluster-net-raw for hints

### Open questions
- How do I verify if the admission controller is enabled? 

# Linkerd's Observability Features
To gain access to Linkerd’s observability features you only need to install the Viz extension:
```
linkerd viz install | kubectl apply -f -
```
For more information on the subject, see https://linkerd.io/2.10/getting-started/#. 
