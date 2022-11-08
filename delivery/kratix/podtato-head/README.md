# Deliver Podtato-Head with Kratix

Kratix focuses on a slightly different level of abstraction than many of the other options in this repository and actually collaborates with each of the other solutions That is to say, Kratix is unopinionated about how you package and deploy your application, and instead focuses on how you manage the complexity of an entire environment.

If Kratix is adding more complexity in addition to the other delivery mechanisms described in this repository, you may wonder why you would want to include it in your ecosystem. In reality, it is not a good fit if you are deploying a single application or do not have a goal of application team self service.

Kratix introduces the concept of Promises as a way to describe the offerings a Platform Engineering team may provide to their Software Engineer customers. While Podtato-Head is currently stateless, it is not a large leap to say that most teams deploying an application like Podtato-Head, will require a few environments for testing, maybe some baseline dashboards for operability, and maybe even some cloud resources like a DNS record, blob storage, or a a cache. Promises are a way for specialist teams to package each of these needs in a way that encouraging minimal configuration by the application team and can be composed into a single, manageable deliverable.

# Prerequisites

## Install Kratix

To get started, you will want to prepare a local environment. There is a easy to use script which only depends on `docker`, `kind`, and `kubectl`.

To use this script, clone the Kratix repo and run the script:
```bash
git clone https://github.com/syntasso/kratix.git
cd kratix
./scripts/quick-start.sh
```

This script creates two local clusters (`platform` and `worker`) and configures both. 

On the platform cluster, this script installs Kratix, [MinIO](https://min.io/) to act as a local GitOps repository, and a Custom Resource that provides as a reference to the worker cluster.

On the worker cluster this script installs [Flux](https://fluxcd.io/) as a GitOps reconciler which reads from the Platform cluster MinIO storage.

> 📝 This multi-cluster scenario is not required, however it is useful to visualise the handover of responsibility between platform and application responsibility.

## Generating a library of Promises

Promises are a way for platform engineers to reduce complexity for application developers while also being able to design intentional standardization.

There are three key parts to a Promise:

1. A Custom Resource Definition (called `xaasCrd`) which defines any requirements and options for the application developer. This can include specific requirements like a name that is DNS friendly.
1. A set of prerequisite resources (called `workerClusterResources`) that can be shared across all instances of a Promise and must be present for any one instance to be created. For example, the team may install a Prometheus operator to help create a monitoring stack when an instance of an application is created.
1. A pipeline (called an `xaasRequestPipeline`) that defines any per-instance configuration. This is where `xaasCrd` specifications would be applied alongside any business defined requirements like encryption.

The easiest way to see this is to review and install a few.

### Install a minimal Promise

> **To install this promise, use:**
> **`kubectl apply -f ./kubectl/kubectl-promise.yaml`**

The most common example of creating things in Kubernetes very well might be `kubectl apply`. For that reason, the first Promise is using the content from the [`kubectl` delivery example](https://github.com/podtato-head/podtato-head/tree/main/delivery/kubectl).

In many ways, this Promise does not add any value. It includes _moar_ running software (Kratix) and only moves the flat YAML files from one location to another.

But, it does start to show some benefits. It shows how you can encapsulate the knowledge of swapping out service type `LoadBalancer` for type `NodePort`, and also logic around unique prefix names.

> 🔜 Notice that for now the two separate functions run by the pipeline are written in different shell scripts but run in the same Docker image. The intention is for each step of the pipeline to be a unique OCI image, but as of now we only support a single image in the pipeline.

### Using user data when delivering a Promise

TODO: Select a different delivery mechanism and showcase a Promise that leverages user data set via the `xaasCrd` spec when generating the output files. Possibly use Kustomize and decide if deploying to dev or prod (with the overlay).

### Delivering supporting infrastructure with a compound Promise

TODO: Select a third delivery mechanism and showcase a Promise that actually delivers two different capabilities. Possibly generate a GitHub Repo and then use that repo to support the Flux delivery.

# Deliver

Once promises exist, users can request an instance of that offered capability by submitting a resource request. In some cases this requires or offers customization fields. In other cases an empty request is enough.

To select one each of the above promises you can request:

```bash
kubectl apply -f ./kubectl/kubectl-resource-request.yaml
```

# Test

## Verify delivery

There are three key levels of delivery.

First, you will want to validate that the platform has offered the expected capabilities. You can see all available promises with:

```bash
kubectl --context kind-platform get promises
```

as well as the expected prerequisite resources which is easiest to see by reviewing namespaces on the worker cluster. There should be one per promise type installed:

```bash
kubectl --context kind-worker get namespaces
```

Next, you may want to know all requests for resources across promises. You can review this with:

```bash

```

and see that the requests were serviced via a Request Pipeline by looking for a pipeline pod for each requested resource:

```bash
kubectl --context kind-worker -n default get pods
```

Then, you will want to be sure that these resource requests resulted in the correct services running by checking pods in the worker cluster:

```bash
kubectl --context kind-worker get services -l app=podtato-head
```

## Test the API endpoint

To connect to the API you'll first need to determine the correct address and port.

If using a LoadBalancer-type service for entry, get the IP address of the load balancer and use port 9000:

```bash
ADDR=$(kubectl get service podtato-entry -n podtato-flux -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
PORT=9000
```

If using a NodePort-type service, get the address of a node and the service's NodePort as follows:

```bash
export NODE_NAME=<any_node_name>
ADDR=$(kubectl get nodes ${NODE_NAME} -o jsonpath={.status.addresses[0].address})
PORT=$(kubectl get services podtato-entry -n podtato-flux -ojsonpath='{.spec.ports[0].nodePort}')
```

If using a ClusterIP-type service, run kubectl port-forward in the background and connect through that:

> NOTE: Find and kill the port-forward process afterwards using ps and kill.

```bash
ADDR=127.0.0.1
PORT=9000
kubectl port-forward --address ${ADDR} svc/podtato-entry ${PORT}:9000 &
```

Now test the API itself with curl and/or a browser:

```bash
curl http://${ADDR}:${PORT}/
xdg-open http://${ADDR}:${PORT}/
```

> 🔜 In the future Promises will service information like the `ADDR` and `PORT` via status fields. This is not yet implemented, however the intent is for platform engineers to surface any useful information back to the software engineers via their shared interface (a Promise).

# Purge

If you created test clusters using the setup script, feel free to delete the clusters using:

```bash
kind delete clusters worker platform
```

If you want to only remove the resources, then deleting a Promise will remove any generated resources.

```bash
kubectl delete -f ./kubectl/kubectl-promise.yaml
```
