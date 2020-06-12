### Introduction:
Helm is a package manager for Kubernetes applications. In this tutorial, we'll understand the basics of Helm and how they form a powerful tool for working with Kubernetes resources. 

### Helm Architecture:

Helm has a fairly simple architecture, which is comprised of a client and an in-cluster server:

**Tiller Server:** Helm manages Kubernetes application through a component called Tiller Server installed within a Kubernates cluster. Tiller interacts with the Kubernetes API server to install, upgrade, query and remove Kubernetes resources.
**Helm Client:** Helm provides a command-line interface for users to work with Helm Charts. Helm Client is responsible for interacting with the Tiller Server to perform various operations like install, upgrade and rollback charts.

### Helm Charts
Helm manages Kubernetes resource packages through Charts.

We'll see more about charts as we create them shortly, but for now, a chart is nothing but a set of information necessary to create a Kubernetes application, given a Kubernetes cluster:

A chart is a collection of files organized in a specific directory structure. The configuration information related to a chart is managed in the configuration. Finally, a running instance of a chart with a specific config is called a release.


### Installing Helm
The quickest way to install helm on mac is through brew package.
```
brew install helm
```
This installs the Helm Client locally. Now, we need to initialize Helm CLI, which effectively also installs the Tiller Server onto a Kubernetes cluster as identified through the Kubernetes configuration. 
Please ensure that the Kubernetes cluster is running and accessible through kubectl before initializing Helm.
```
kubectl cluster-info
```
Heml CLI should be initialized by default.

### Creating Helm chart
We will initiate our first chart :
```
helm create hello-world
```
Please note that the name of the chart provided here will be the name of the directory where the chart is created and stored. 