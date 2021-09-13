# Appendix
### Word on PodSecurityPolicies on GKE
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