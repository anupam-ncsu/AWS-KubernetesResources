# Istio - An Introduction to service mesh and its dashboard

![CARTOON](/_temp/istio-header.jpeg)

## What is a service mesh and why do we need it ?
The term service mesh is used to describe the network of microservices that make up such applications and the interactions between them. As a service mesh grows in size and complexity, it can become harder to understand and manage. Its requirements can include discovery, load balancing, failure recovery, metrics, and monitoring. A service mesh also often has more complex operational requirements, like A/B testing, canary releases, rate limiting, access control, and end-to-end authentication.

## what is Istio
Istio service mesh is a **sidecar container implementation** of the features and functions needed to provide **behavioral insights and operational control** over the microservices cluster as a whole. We get these benefits with **no changes to our source code.**   
By using the sidecar model, Istio runs in a Linux container in our Kubernetes pods (much like a sidecar rides along side a motorcycle) and injects and extracts functionality and information based on our configuration that lives outside of our code decreasing complexity and heft. It also **moves operational aspects away from code development** and into the domain of operations.

Istio's functionality running outside of our source code introduces the concept of Service Mesh. That's a coordinated group of one or more binaries that make up a **mesh of networking functions** such as:   

> **Traffic Management:** Control the flow of traffic and API calls between services, make calls more reliable, and make the network more robust in the face of adverse conditions. 

> **Observability:** Gain understanding of the dependencies between services and the nature and flow of traffic between them, providing the ability to quickly identify issues.

> **Policy Enforcement** Apply organizational policy to the interaction between services, ensure access policies are enforced and resources are fairly distributed among consumers. Policy changes are made by configuring the mesh, not by changing application code.

> **Service Identity and Security:** Provide services in the mesh with a verifiable identity and provide the ability to protect service traffic as it flows over networks of varying degrees of trustability.

## Istio Architecture

An Istio service mesh is logically split into a **data plane and a control plane.**

The **data plane** is composed of a set of intelligent **Envoy proxies** deployed as sidecars. These proxies mediate and control all network communication between microservices along with **Mixer**, a general-purpose policy and telemetry hub.

The **control plane** manages and configures the proxies to route traffic. Additionally, the control plane configures Mixers to enforce policies and collect telemetry.

The following diagram shows the different components that make up each plane: 

![CARTOON](/_temp/istio-arch.png)

- **Envoy:** Mediate all inbound and outbound traffic for all services in the service mesh.  
- **Mixer:** Enforces access control and usage policies across the service mesh, and collects telemetry data from the Envoy proxy.  
- **Pilot:** Pilot provides service discovery for the sidecars, traffic management capabilities for intelligent routing (e.g., A/B tests, canary deployments, etc.), and resiliency (timeouts, retries, circuit breakers, etc.). It converts high level routing rules into Envoy-specific configurations, and propagates them to the sidecars at runtime.
- **Citadel:** Citadel provides strong service-to-service and end-user authentication with built-in identity and credential management. 
----
## Getting started

For this setup I am using a local kubernetes cluster on my machine. Docker Desktop now comes with a kubernetes setup. I expect you will have the setup ready with a kubectl.

Once we have started the kubernetes instances, we create a **namespace : istio-system** and start all of the Istio-related components. From there, as we create projects and pods, istio adds configuration information to our deployments, and our pods will use Istio. 

Istio is installed in two parts. The first part involves the CLI tooling that will be used to deploy and manage Istio backed services. The second part configures the Kubernetes cluster to support Istio.


### Install CLI tooling
The following command will download the Istio 1.0.0 release on your local.
```
curl -L https://istio.io/downloadIstio | sh -
```
point the path to the destination folder
```
cd istio-1.5.2
export PATH=$PWD/bin:$PATH
```

### Configure Istio
For this blog, we will apply the **demo** configuration profiles.  
The profiles provide customization of the Istio control plane and of the sidecars for the Istio data plane. You can start with one of Istio’s built-in [configuration profiles](https://istio.io/docs/setup/additional-setup/config-profiles/) and then further customize the configuration for your specific needs. 

For this demo, please fork the files from [here](https://github.com/anupam-ncsu/AWS-KubernetesResources/tree/master/Istio/manifest)

```
istioctl manifest apply --set profile=demo
```
- Add a [namespace](https://github.com/anupam-ncsu/AWS-KubernetesResources/blob/master/Istio/manifest/namespace-dev.yaml) label to instruct Istio to automatically inject Envoy sidecar proxies when you deploy your application later. In this case, every app that gets deployed to **namespace : dev** will have istio enabled.
```
kubectl apply - f manifest/namespace-dev.yaml
kubectl label namespace dev istio-injection=enabled
```
### Deploy Applications
For simplicity, we will use the famous [bookinfo sample published by the istio team with minor modifications.](https://github.com/anupam-ncsu/AWS-KubernetesResources/blob/master/Istio/manifest/bookinginfo-dev.yaml)
```
kubectl apply -f manifest/bookinginfo-dev.yaml
```
The application will start. As each pod becomes ready, the Istio sidecar will deploy along with it.

```
kubectl get services -n dev

kubectl get pods - n dev
```
Wait till the pods are in ready state  
Verify everything is working correctly up to this point. Run this command to see if the app is running inside the cluster and serving HTML pages by checking for the page title in the response:
```
kubectl exec -it $(kubectl get pod -l app=ratings -n dev -o jsonpath='{.items[0].metadata.name}') -n  dev -c ratings -- curl productpage:9080/productpage | grep -o "<title>.*</title>"

<title>Simple Bookstore App</title>
```
### Istio Ingress: Open application to traffic
The Bookinfo application is deployed but not accessible from the outside.   

To make it accessible, you need to create an [Istio Ingress Gateway](https://github.com/anupam-ncsu/AWS-KubernetesResources/blob/master/Istio/manifest/bookinginfo-ingress.yaml), which maps a path to a route at the edge of your mesh. Gateway configurations are applied to standalone Envoy proxies that are running at the edge of the mesh, rather than sidecar Envoy proxies running alongside your service workloads.  
Gateway configurations are applied to standalone Envoy proxies that are running at the edge of the mesh, rather than sidecar Envoy proxies running alongside your service workloads. Unlike other mechanisms for controlling traffic entering your systems, such as the Kubernetes Ingress APIs, Istio gateways let you use the full power and flexibility of Istio’s traffic routing. You can do this because Istio’s Gateway resource just lets you configure layer 4-6 load balancing properties such as ports to expose, TLS settings, and so on. Then instead of adding application-layer traffic routing (L7) to the same API resource, you bind a regular Istio virtual service to the gateway. This lets you basically manage gateway traffic like any other data plane traffic in an Istio mesh.

```
kubectl apply -f bookinginfo-ingress.yaml
```
Confirm that the gateway got created
```
kubectl get gateway -n dev
```

### Ingress IP and Ports
Check the ingress gateway External IP
```
kubectl get svc istio-ingressgateway -n istio-system
NAME                   TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                                                                                                                                      AGE
bookinfo-gateway   LoadBalancer   10.111.214.48   localhost     15020:30172/TCP,80:30126/TCP,443:30000/TCP,15029:30787/TCP,15030:30922/TCP,15031:31764/TCP,15032:31819/TCP,31400:31537/TCP,15443:31256/TCP   17h
```
If the EXTERNAL-IP value is set, your environment has an external load balancer that you can use for the ingress gateway. If the EXTERNAL-IP value is <none> (or perpetually <pending>), your environment does not provide an external load balancer for the ingress gateway. In this case, you can access the gateway using the service’s node port.

As I am running the kubernetes cluster on my local , the hostname is:
```
127.0.0.1
```
The port is the one we specified in our gateway defination: 80
The apps are exposed through the VirtualService which links to the app **productpage** . 

## Microservice structure visualization
We can visualize the app through the istio default dashboard on kiali.
Open a new terminal instance and execute:
```
istioctl dashboard kiali
```
the visualization is simple to understand and debug.

![CARTOON](/_temp/kiali.png)

Notice, that we linked only to the product page through the ingress virtual service. The Rest of the mesh is connected through their own service definitions.
