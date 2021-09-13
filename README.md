# Setup
## Setting up our Kubernetes Cluster
Before installing Linkerd we need an underlying Kubernetes Cluster. If you don't have one, you can create on with the following steps:

1. Enable the Google Kubernetes Engine API.
    ```
    gcloud services enable container.googleapis.com
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

We will also be installing a nginx ingress controller and to prepare for this, we need a static IP address. If you don't have one already:
1. in the Google Cloud Console, go to _VPC Network_ -> _External IP addresses_, then click on _Reserve Static Address_. You can keep all the default settings. 

### Installing Linkerd
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
    If you get error message on `NET_ADMIN` and `NET_RAW`, you can ignore it for this demo. It is caused by a GKE limitation, more details are found in the appendix. 
5. Install the Linkerd control plane. This will install the control plane into the `linkerd` namespace.
    ```
    linkerd install | kubectl apply -f - 
    ```
6. Validate that everything worked
    ```
    linkerd check
    ```



### Installing viz
To gain access to Linkerd’s observability features, you will be using a Linkerd extension called Viz (steps taken from https://linkerd.io/2.10/features/telemetry/:
1. Install the extension into your Kubernetes cluster, in the `linkerd-viz` namespace.
    ```
    linkerd viz install | kubectl apply -f -
    ```
2. Verify what was installed
    ```
    kubectl get all -n linkerd-viz
    ```
3. The verify that it worked, open the dashboard created by viz
    ```
    linkerd viz dashboard &
    ```

### Installing the demo application
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
3. To verify that it works, run 
    ```
    kubectl -n emojivoto port-forward svc/web-svc 8080:80
    ```
    and visit the web preview in your cloud shell. 

### Installing IngressController and Ingress
To access the viz dashboard and the demo app from a fixed DNS, we will be setting up an ingress controller and ingresses for the dashboard and demo app. Since Linkerd does not come with an ingress controller out of the box, we'll be using it with ingress-nginx. 
1. Install the IngressController using Helm
    ```
    helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
    helm repo update
    helm pull ingress-nginx/ingress-nginx --version 3.35.0
    tar -xvzf  ingress-nginx-3.35.0.tgz
    cd ingress-nginx
    ```
2. We will inject our static IP address to the ingress controller using a `custom_values.yml` file. 
    ```
    cat <<EOF> custom_values.yml
    controller:
        admissionWebhooks:
            enabled: false
    service:
        loadBalancerIP: XX.XX.XX.XX
    metrics:
        enabled: true
    EOF
    ```
3. Install the helm chart with the custom values
    ```
    helm install ingress-nginx . --values custom_values.yaml --version 3.35.0
    ```
4. In `manifests/dashboard-ingress.yml` and `manifests/emojivoto-ingress.yml`, your can find routing rules the dashboard and the demo app, respectively. 
    ```
    spec:
        ingressClassName: nginx
        rules:
            - host: dashboard.hilmaja.com
              http:
              paths:
                - path: /
    ```
    If you want to follow along, make sure to change it to your own domain and add two A-records that point to the loadbalancer ingress IP set above.
5. Install the Ingress for the Linkerd dashboard and the emojivoto app
    ```
    kubectl apply -f manifests/dashboard-ingress.yml
    kubectl apply -f manifests/emojivoto-ingress.yml
    ```
6. Verify that the resources can be accessed at http://dashboard.yourdomain.com and http://emojivoto.yourdomain.com

# Observability
### Golden Metrics
- Success Rate

```
linkerd viz routes service/voting-svc -n emojivoto
```

- Traffic (Requests per second)
- Latencies 

### Debugging
- https://linkerd.io/2.10/tasks/debugging-your-service/

# Cleaning up
```
gcloud container clusters delete linkerd-observability \
    --region us-central1
```

