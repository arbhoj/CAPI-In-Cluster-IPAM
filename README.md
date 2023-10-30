# CAPI-In-Cluster-IPAM
This repo provides an example for using the new [In-Cluster IPAM](https://github.com/kubernetes-sigs/cluster-api-ipam-provider-in-cluster) feature for [CAPV](https://github.com/kubernetes-sigs/cluster-api-provider-vsphere) that provides an option to use pre-defined IPs for nodes being providened. 

> Note:
> - This requires CAPV 1.6.0+ as the older versions don't support it.
> - The CAPI Mangement cluster should already have [cert-manager](https://github.com/cert-manager/cert-manager) running on it. 


This is very useful for scenarios such as
- vSphere environments without DHCP
- Requirement to pre-define a VIP in an external load-balancer for Control Plane HA, which would require the control plane node IPs to be populated in the backend list for that VIP

Here is an example on how this works:
> Note: These steps need to be performed on the CAPI Management cluster which already have CAPV 1.6.0+ and cert-manager components running as mentioned earlier.

Step 1: Deploy the In-Cluster IPAM components
```
kubectl apply -f https://github.com/kubernetes-sigs/cluster-api-ipam-provider-in-cluster/releases/download/v0.1.0-alpha.3/ipam-components.yaml
``````

Step 2: Create the InClusterIPPool resource for a cluster

```
kubectl apply -f - <<EOF
---
apiVersion: ipam.cluster.x-k8s.io/v1alpha1
kind: InClusterIPPool
metadata:
  name: dkp-demo-pool
  namespace: default
spec:
  prefix: 20
  gateway: 10.16.48.1
  addresses:
  - 10.16.56.88
  - 10.16.56.89
  - 10.16.56.90
  - 10.16.56.91
  - 10.16.56.92
  - 10.16.56.93
EOF
```

Step 3: Create a Cluster definition with a reference to the Pool created in the VSphereMachineTemplate resources in it’s network section and deploy the cluster
e.g.:
```
     network:
       devices:
       - dhcp4: false
         networkName: vms
         nameservers:
         - 10.16.48.1
         - 8.8.8.8
         - 8.8.4.4
         addressesFromPools:
         - apiGroup: ipam.cluster.x-k8s.io
           kind: InClusterIPPool
           name: dkp-demo-pool
```

The cluster will be deployed with the nodes getting IPs defined in the pool instead of DHCP.

> Note: The following steps only apply if the cluster is going to be made self managed. 

Step 4: Deploy the In-Cluster IPAM components

```
kubectl apply -f https://github.com/kubernetes-sigs/cluster-api-ipam-provider-in-cluster/releases/download/v0.1.0-alpha.3/ipam-components.yaml
```

Step 5: Move the capi-resources from the bootstrap cluster to the workload cluster to make it self managed. The controller will automatically sync IP Addresses based on the “ipaddressclaims” resource which gets generated automatically.

```
kubectl get ipaddressclaims
```

Step 6: Recreate the InClusterIPPool resource and then check it’s status. Also check that “ipaddress” resources for the allocated IPs are now automatically created.

```
kubectl apply -f - <<EOF
---
apiVersion: ipam.cluster.x-k8s.io/v1alpha1
kind: InClusterIPPool
metadata:
  name: dkp-demo-pool
  namespace: default
spec:
  prefix: 20
  gateway: 10.16.48.1
  addresses:
  - 10.16.56.88
  - 10.16.56.89
  - 10.16.56.90
  - 10.16.56.91
  - 10.16.56.92
  - 10.16.56.93
EOF
```

```
kubectl get ipaddress
```

Step 7: Perform cluster upgrades, updates, scaling etc. The pool will work as expected.





