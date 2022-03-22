# AWS App Mesh - because managing microservices at scale is hard

This section assumes that you have completed the previous section named **"Deploy Backend Services"**.
The assumptions listed in that section also apply here.

Install the [App Mesh Controller](https://aws.github.io/aws-app-mesh-controller-for-k8s/) as follows, ignoring any warnings.
```bash
eksctl create iamserviceaccount \
  --cluster ${EKS_CLUSTER_NAME} \
  --namespace kube-system \
  --name appmesh-controller \
  --attach-policy-arn arn:aws:iam::aws:policy/AWSCloudMapFullAccess,arn:aws:iam::aws:policy/AWSAppMeshFullAccess \
  --override-existing-serviceaccounts \
  --approve

helm repo add eks https://aws.github.io/eks-charts
helm -n kube-system upgrade -i appmesh-controller eks/appmesh-controller \
  --set region=${AWS_DEFAULT_REGION} \
  --set serviceAccount.create=false \
  --set serviceAccount.name=appmesh-controller
```

Verify that the App Mesh Controller is installed.
```bash
kubectl -n kube-system get deployment appmesh-controller
```

**If not already in place**, in a **dedicated** terminal window run a looped command against the **frontend**.
```bash
kubectl exec -it jumpbox -- /bin/bash -c "while true; do curl http://echo-frontend-blue.demos.svc.cluster.local:80; sleep 0.25; done"
```

The next step is to start rolling out the [AWS App Mesh components](https://docs.aws.amazon.com/app-mesh/latest/userguide/what-is-app-mesh.html#app_mesh_components).
Go the the [App Mesh console](https://us-west-2.console.aws.amazon.com/appmesh/meshes) page.
There is likely to be no Meshes currently shown here.
Each `Mesh` resource encapsulates a logical collection of other interconnected service mesh resources, as revealed shortly.

Download the manifest for your `Mesh` resource to your Cloud9 environment.
```bash
mkdir -p ~/environment/mesh/templates/
wget https://raw.githubusercontent.com/${EKS_GITHUB_USER}/eks-demos/main/mesh/templates/demos-mesh.yaml \
  -O ~/environment/mesh/templates/demos-mesh.yaml
```

Introduce a `Chart.yaml` file so your mesh is deployable via Helm
```bash
cat > ~/environment/mesh/Chart.yaml << EOF
apiVersion: v2
name: mesh
version: 1.0.0
EOF
```

Use the Helm CLI to deploy your service mesh in its embryonic state.
The mesh will be fleshed out, step by step, in a manner which respects the referential integrity of its resources so you will repeat this step a few more times.
```bash
# creating Mesh
helm -n demos upgrade -i mesh ~/environment/mesh
```

As the `Mesh` object is introduced to the Kubernetes cluster the App Mesh Controller reacts by building the new AppMesh **Mesh** resource in your AWS account.
Use the following commands to view these **twinned** objects/resources.
```bash
# K8s - NOTE unlike VirtualX objects, Mesh objects are not namespaced
kubectl get meshes 
kubectl describe mesh demos
# AWS
aws appmesh list-meshes
aws appmesh describe-mesh --mesh-name demos
```

Return to the App Mesh console to observe your nascent "Mesh" resource named `demos` which you can click through to see its (empty) configuration.
You must **resist** any temptation to create or modify meshes via the console.
That's the implicit agreement you make whenever you use Kubernetes manifests/controllers to "manage" external resources in this way.
Obviously it's convenient to **view** your configuration via the console, but when using the App Mesh Controller you must respect this arrangement and treat the console as if it were **read-only**.

<!-- The console reveals that your mesh encompasses, among other things, `Virtual nodes` of which there are currently none. -->

Labels in Kubernetes are much like tags in AWS.
They can be used to provide (automated) observers the required context to behave as intended.
For namespaces to support AppMesh they first need to be associated with their `Mesh` object.
This association is activated with a namespace label.

Additionally, resident pods will need to be "proxied" for use with your service mesh.
We could do this by manually adding the [envoy](https://www.envoyproxy.io/) service proxy container to every deployment manifest.
Thankfully, the use of the injector webhook, which is installed with the App Mesh Controller, prevents us from needing to do this explicitly.
The injector webhook is also activated with a label.

Use a pair of labels to activate your namespace for use with your mesh by applying a pair of labels as follows.
```bash
kubectl label namespace demos \
  mesh=demos \
  appmesh.k8s.aws/sidecarInjectorWebhook=enabled
```

Each Kubernetes service which wants to play a role in your service mesh requires an associated `VirtualNode` resource inside the mesh.
Your **backend** `VirtualNodes` are the simplest to implement since they are not dependent on any other `Virtual` objects inside the mesh itself.

Download the manifests for your `VirtualNode` **backend** resources.
```bash
wget https://raw.githubusercontent.com/${EKS_GITHUB_USER}/eks-demos/main/mesh/templates/vn-echo-backend-blue.yaml \
  -O ~/environment/mesh/templates/vn-echo-backend-blue.yaml
wget https://raw.githubusercontent.com/${EKS_GITHUB_USER}/eks-demos/main/mesh/templates/vn-echo-backend-green.yaml \
  -O ~/environment/mesh/templates/vn-echo-backend-green.yaml
```

Use the Helm CLI to redeploy your service mesh with the new `VirtualNode` **backend** resources now in place.
```bash
# adding backend VirtualNodes
helm -n demos upgrade -i mesh ~/environment/mesh
```

View the new objects/resources.
```bash
# K8s
kubectl -n demos get virtualnodes
kubectl -n demos describe virtualnode vn-echo-backend-blue
kubectl -n demos describe virtualnode vn-echo-backend-green
# AWS
aws appmesh list-virtual-nodes --mesh-name demos
aws appmesh describe-virtual-node --mesh-name demos \
  --virtual-node-name vn-echo-backend-blue
aws appmesh describe-virtual-node --mesh-name demos \
  --virtual-node-name vn-echo-backend-green
```

Your primary aim is to dynamically orchestrate the distribution of traffic from the frontend to both your **blue** and **green** backends.
In this case, App Mesh requires a `VirtualRouter` resource to sit just in front of your backend `VirtualNodes` and maintain the desired traffic split ratios.

Download the manifests for your `VirtualRouter` resource.
```bash
wget https://raw.githubusercontent.com/${EKS_GITHUB_USER}/eks-demos/main/mesh/templates/vr-echo-backend.yaml \
  -O ~/environment/mesh/templates/vr-echo-backend.yaml
```

Use the Helm CLI to redeploy your service mesh with the new backend `VirtualRouter` resource now in place.
Note the use of templated values for the split ratios.
```bash
# adding VirtualRouter - note the percentage blue/green weights
helm -n demos upgrade -i mesh ~/environment/mesh \
  --set weightBlue=100 \
  --set weightGreen=0
```

View the new objects/resources.
```bash
# K8s
kubectl -n demos get virtualrouters
# AWS
aws appmesh list-virtual-routers --mesh-name demos
aws appmesh list-routes --mesh-name demos \
  --virtual-router-name vr-echo-backend
aws appmesh describe-route --mesh-name demos \
  --virtual-router-name vr-echo-backend \
  --route-name vrr-echo-backend
```

 That final step is where you can observe the weights.
 Return to the App Mesh console and see if you can locate the weights there.

Your next step is to introduce a `VirtualService` object which depends upon the `VirtualRouter` object you just created as well as an underlying Kubernetes `Service` object.
The `Service` you twin with your `VirtualService` intentionally missing a `spec:selector:` section which means it can never target any traditional pod endpoints which is intentional.
It just needs to surface an IP address for identity purposes.

Download the manifests for your `VirtualService` resource and its associated `Service` object.
```bash
wget https://raw.githubusercontent.com/${EKS_GITHUB_USER}/eks-demos/main/mesh/templates/vs-echo-backend.yaml \
  -O ~/environment/mesh/templates/vs-echo-backend.yaml
wget https://raw.githubusercontent.com/${EKS_GITHUB_USER}/eks-demos/main/mesh/templates/vs-echo-backend-service.yaml \
  -O ~/environment/mesh/templates/vs-echo-backend-service.yaml
```

Use the Helm CLI to redeploy your service mesh with the new `VirtualService` resource and associated `Service` object now in place.
```bash
# adding VirtualService
helm -n demos upgrade -i mesh ~/environment/mesh \
  --set weightBlue=100 \
  --set weightGreen=0
```

View the new resources/objects.
```bash
# K8s
kubectl -n demos get virtualservices
kubectl -n demos get services
kubectl -n demos get endpoints
# AWS
aws appmesh list-virtual-services --mesh-name demos
aws appmesh list-routes --mesh-name demos \
  --virtual-router-name vr-echo-backend
aws appmesh describe-route --mesh-name demos \
  --virtual-router-name vr-echo-backend \
  --route-name vrr-echo-backend
```

The `VirtualNode` **frontend** resource was deferred until now since it has a dependency on your `VirtualService` resource.
Now that dependency is in place we can now complete the Mesh

Download the manifest for your `VirtualNode` **frontend** resource.
```bash
wget https://raw.githubusercontent.com/${EKS_GITHUB_USER}/eks-demos/main/mesh/templates/vn-echo-frontend-blue.yaml \
  -O ~/environment/mesh/templates/vn-echo-frontend-blue.yaml
```

Use the Helm CLI to redeploy your **completed** service mesh with the new `VirtualNode` **frontend** resource now in place.
```bash
# adding frontend VirtualNode
helm -n demos upgrade -i mesh ~/environment/mesh \
  --set weightBlue=100 \
  --set weightGreen=0
```

View the new objects/resources.
```bash
# K8s
kubectl -n demos get virtualnodes
kubectl -n demos describe virtualnode vn-echo-frontend-blue
# AWS
aws appmesh list-virtual-nodes --mesh-name demos
aws appmesh describe-virtual-node --mesh-name demos \
  --virtual-node-name vn-echo-frontend-blue
```

With the Mesh now complete and deployed you can restart all your backend deployments to get the envoy proxies injected and configured
Observe this happening, from **another dedicated** terminal window, as follows.
```bash
# ctrl+c to quit
watch kubectl -n demos get pods
```

Now bounce the **backend** deployments and watch them return with one new container per pod.
```bash
kubectl -n demos rollout restart deployment \
  echo-backend-blue \
  echo-backend-green
```

Almost there.
Finally, we need to bounce the **frontend** deployment but, as you do, take this opportunity to reconfigure the backend URL to point at our "meshed" service.
```bash
helm -n demos upgrade -i echo-frontend-blue ~/environment/echo-frontend/ \
  --set registry=${EKS_ECR_REGISTRY} \
  --set color=blue \
  --set version=2.0 \
  --set backend=http://vs-echo-backend:80 \
  --set serviceType=ClusterIP
```

You should still have open a **dedicated** terminal window polling the frontend and, at this point, nothing appears to have changed because we weighted the `VirtualRouter` to send 100% of traffic to the `blue` backend.
Let's reconfigure the `VirtualRouter` to split the traffic 50/50 and observe what happens after ~10 seconds have elapsed.
```bash
helm -n demos upgrade -i mesh ~/environment/mesh \
  --set weightBlue=50 \
  --set weightGreen=50
```

This is a textbook-style blue/green configuration with zero impact on upstream services.
But we've only scratched the surface of what's possible.
Service meshes are all about externalizing the type of logic that you don't want polluting your codebase.
Other use cases include.
- Observability
- Retry policies
- TLS termination
- Circuit breakers

You may hear these topics collectively described as **cross-cutting concerns** or **non-functional requirements** - that is to say businesses rarely ask for these features explicitly, but they may question their absence when things go wrong.

Whilst a service mesh may appear overkill in the small problem domain presented here, they become ever more important as your microservices architecture grows.

[Return To Main Menu](/README.md)