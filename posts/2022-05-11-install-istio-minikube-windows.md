# How to install Istio on a Minikube cluster on Windows

## Background
[Kube by Example](kubebyexample.com) has some great guides to help people to learn Kubernetes but they assume you're running Linux, which I don't always do. Having worked through their exercise on building Istio on Minikube on Windows 11 I thought I would write up my own guide to refer to in the future.

## Aknowledgements
This guide is heavily based on the [Installing Istio on a Minikube Cluster](https://kubebyexample.com/en/learning-paths/istio/install) guided exercise on [Kube by Example](kubebyexample.com). Thanks to them for the excellent learning material.

## Outcomes
After completing this guide you will have:
- An instance of Istio installed on a Minikube cluster.
- Identified the ingress service endpoint for Istio.
- Deployed the Istio add-ons.
- An example application running to test the Istio installation.

## Pre-requisites
This guide assumes that you have `kubectl` and `minikube` installed and are using PowerShell to run commands.

As we're deploying Istio on Minikube and not a public cloud we will use [MetalLB](https://metallb.universe.tf/) to provide a network load-balancer implementation for the cluster. Follow [this guide](https://tonejito.github.io/kbe/topics/metallb/install/) to install MetalLB on a Minikube cluster. It doesn't require any changes to work on Windows.

## Instructions
1. Start `minikube`, ensuring that the cluster meets the minimum requirements for running Istio.
```
minikube start --cpus=4 --memory=8g
```

2. Check that the pods in the `metallb-system` namespace are running.
```
kubectl get pods -n metallb-system
```

3. Verify that MetalLB has the correct IP address range configured. If you're not sure, go back and check the [MetalLB installation guide]((https://tonejito.github.io/kbe/topics/metallb/install/)). 
```
kubectl get configmap config -n metallb-system -o yaml
```

4. Download the [latest release of Istio](https://github.com/istio/istio/releases/) from Github and extract the .zip file into a location of your choosing.

5. Add the location of the `istioctl` application to `PATH`.

6. Run a pre-check to test whether the cluster meets the requirements to install Istio.
```
istioctl experimental precheck
```

7. Install Istio on the cluster.
```
istioctl install --set profile=demo -y
```

8. List the resources created in the `istio-system` namespace.
```
kubectl get deployments,pods -n istio-system
```

9. To be continued...
