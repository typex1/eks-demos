# K8s LoadBalancer Services - because the world needs to talk to our cluster

The NodePort service solved one problem but exposed another.
If our nodes (i.e. EC2 instances) belong to a functioning auto-scaling group then we are faced with some important questions:

- How many nodes are currently active and can I distribute the traffic evenly among them?
- What are their IP addresses and are they reachable by your clients?
- What if the node we were targeting a moment ago has now been terminated?

In general, load balancers are designed to negate these questions by providing an active single point of access which guards us from the underlying complexity.
Kubernetes services of type LoadBalancer incorporate and extend the functionailty of NodePort services which, in turn, extend the functionality of the ClusterIP services.
In EKS, these services provide access to the underlying NodePort service via an [AWS Classic Load Balancer](https://aws.amazon.com/elasticloadbalancing/classic-load-balancer).

Kubernetes clusters provide extensibilty via pluggable components known as [controllers](https://kubernetes.io/docs/concepts/architecture/controller/) which can be customized for a variety of purposes.
Controllers take their instructions from type-matched Kubernetes objects when they are applied to the cluster.
When newly created objects appear within a cluster the associated controller(s) will react by taking whatever action necessary to reconcile.
It is the same pattern of behaviour we see for all native Kubernetes objects so this idea of reconciliation (i.e. desired vs. actual) should feel familiar.
The reconciliation actions may require the controller to reach out beyond the cluster and into the cloud provider environment itself.
This pattern helps the developer remain focused on one set of tools (i.e. Kubernetes manifests) to deploy both their applications and the wider set of associated infrastructure resources.
Services of type LoadBalancer in EKS are a prime example of the use of custom controllers.

Upgrade the NodePort service to a LoadBalancer service, then check the services.
```bash
kubectl -n ${EKS_APP_NS} get services
kubectl -n ${EKS_APP_NS} patch service ${EKS_APP} --patch '{"spec": {"type": "LoadBalancer"}}' # extend the service to expose a new AWS Load Balancer
sleep 5 && kubectl -n ${EKS_APP_NS} get services
```

The `EXTERNAL-IP`, which was previously set as `<none>`, now contains the publicly addressible DNS name for an AWS Load Balancer.
Port 80 requests arriving at this endpooint are now even distributed across all worker nodes using their node port as before.
Grab the load balancer DNS name and put the following `curl` command in a loop as the AWS resource will not be immediately resolved (2-3 mins).
If you receive any errors, just wait a little longer.
```bash
clb_dnsname=$(kubectl -n ${EKS_APP_NS} get service ${EKS_APP} -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
while true; do curl http://${clb_dnsname}; sleep 0.25; done
# ctrl+c to quit loop
```

It is important to recognise the shared DNA that runs through the Kubernetes service types.
Services of type **LoadBalancer** descend from **NodePort** services which, in turn, descend from **ClusterIP** services.

[Return To Main Menu](/README.md)