# Setting up a Kubernetes Cluster for the lab
Before installing Linkerd we need access to a Kubernetes Cluster. 
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
To install Linkerd on your environment, follow the instructions below (instructions can be found https://linkerd.io/2.10/getting-started/#): 

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

# Word on PodSecurityPolicies on GKE
Part of the pre-check on my GKE cluster failed with the following messages:

- `‼ has NET_ADMIN capability`: found 1 PodSecurityPolicies, but none provide NET_ADMIN, proxy injection will fail if the PSP admission controller is running
see https://linkerd.io/2.10/checks/#pre-k8s-cluster-net-admin for hints
   
- `‼ has NET_RAW capability`: found 1 PodSecurityPolicies, but none provide NET_RAW, proxy injection will fail if the PSP admission controller is running
see https://linkerd.io/2.10/checks/#pre-k8s-cluster-net-raw for hints

To see what this was about, I had to check the Kubernetes documentation and Linkerd's github. First, some definitions: 

`Pod security policy control`: implemented as an optional admission controller. PodSecurityPolicies are enforced by enabling the admission controller, but doing so without authorizing any policies will prevent any pods from being created in the cluster"

`Admission Controller`: An admission controller is a piece of code that intercepts requests to the Kubernetes API server prior to persistence of the object, but after the request is authenticated and authorized. 

On Github (https://github.com/linkerd/linkerd2/issues/3494):
```
When you first linkerd check --pre, you didn't have the Linkerd control plane installed. But there are some existing pod security policies on your cluster, and none of them have the NET_ADMIN and NET_RAW capabilities.

You can confirm that with kubectl describe psp.

Then you installed the Linkerd control plane with linkerd install.
The installation also installed the Linkerd PSP which have the NET_ADMIN and NET_RAW capabilities.
```
The GKE cluster has a PSP called `gce.gke-metrics-agent`, and `kubectl describe psp` shows that `Allowed Capabilities: <none>`. 

# Install viz to your cluster
To gain access to Linkerd’s observability features you need to install a Linkerd extension called Viz (steps taken from https://linkerd.io/2.10/features/telemetry/:
1. Install the extension into your Kubernetes cluster
    ```
    linkerd viz install | kubectl apply -f -
    ```
2. Verify what was installed
    ```
    kubectl get all -n linkerd-viz
    ```
3. Open the dashboard created by viz
    ```
    linkerd viz dashboard &
    ```

# Install demo application
1. Install emojivoto into the emojivoto namespace by running:
    ```
    curl -sL run.linkerd.io/emojivoto.yml | kubectl apply -f -
    ```
2. Add Linkerd to the demo application by running
    ```
    kubectl get -n emojivoto deploy -o yaml \
  | linkerd inject - \
  | kubectl apply -f -
    ```
3. To see the emojivoto application "in action", do 
    ```
    kubectl -n emojivoto port-forward svc/web-svc 8080:80
    ```
    and visit the web preview in your cloud shell. 

# Install IngressController and Ingress
1. Helm
    ```
    helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
    helm repo update
    helm pull ingress-nginx/ingress-nginx --version 3.35.0
    tar -xvzf  ingress-nginx-3.35.0.tgz
    ```
2. Make a `custom_values.yml` file to be used 
    ```
    controller:
        admissionWebhooks:
            enabled: false
    service:
        loadBalancerIP: XX.XX.XX.XX
    metrics:
        enabled: true
    ```
3. Install the helm chart with the custom values
    ```
    helm install ingress-nginx ingress-nginx/ingress-nginx --values custom_values.yaml --version 3.35.0
    ```
4. Install the Ingress for the Linkerd dashboard
    ```
    kubectl install -f manifests/dashboard-ingress.yml
    kubectl install -f manifests/emojivoto-ingress.yml
    ```

# Final setup
In my case, the resources can be found at
- `dashboard.hilmaja.com`
- `emojivoto.hilmaja.com`