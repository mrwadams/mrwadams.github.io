---
layout: post
title: "Notes from Containers from the Couch | Container Day"
date: 2022-05-17 00:00:00 -0000
categories: kubernetes aws security
---

The following are some notes I made while watching Lukonde Mwila's informative talk on [Container Security in AWS Container Services](https://youtu.be/Iyp9Ugk9oRs?t=4772), which was part of Containers from the Couch Container Day.

## Cluster Weaknesses & Vulnerabilities
![Cluster Weaknesses & Vulnerabilities](/assets/images/cluster-weaknesses-vulns.png)

Security considerations at each layer of the K8S stack:

|Layer  | Security Consideration   |
|-------|-------------|
|Workload | Tends to be top of mind when people think about securing Kubernetes but it's important to consider layers that are lower down in the stack as well.  |
| Control Plane | <ul><li>Need to secure K8S API Server.</li><li>Lots of network traffic between API Server, Scheduler and Controller Manager that also needs to be secured.</li><li>Access to Kubelet API needs to be secured.</li></ul> |
|Infrastructure / Hosts | <ul><li>Only open required ports</li><li>Secure host operating system</li><li>Tools like seccomp, SELinux and AppArmor can help with securing host OS</li><li>[This video](https://www.youtube.com/watch?v=ZRo9sIykO98) gives an example of how to use AppArmor to secure K3S cluster hosts.</li></ul>

## Considerations for securing K8S
![Securing K8S](/assets/images/securing-k8s.png)
- Kubernetes can't currently keep track of identities so you need to manage that outside of the system and then map that to the context of Kubernetes
- ABAC = Attribute Based Access Control
- PSP = Pod Security Policies (these are being deprecated)

## Image Scanning & Policy Enforcement
Scanning only provides point-in-time assurance that a container image is "good". Therefore need to think about scanning at multiple stages in the image lifecycle, not just at creation. Key stages are:
- Container image scanning at CI pipeline stage
    - Tools for this include NeuVector and Trivy
- Registry scanning of images
- Policy enforcement in pipelines
    - Tools for this include Kyverno / Datree

## Secrets Management
- Secrets in Kubernetes are natively insecure. They aren't encrypted at rest, only base64 encoded!
- Bitnami Sealed Secrets solves this issue by providing a mechanism to encrypt the secret data and make it safe to store in Git repositories.
    - Uses `kubeseal` (a CLI tool) to transform Custom Resource Definition (CRD) secrets into sealed secrets.
    - A Sealed Secrets controller that's used to generate the encryption key, and decrypt sealed secrets into secrets that can be used by pods.

### External Secrets Operator (ESO)
- A Kubernetes operator that enables integration with an external secrets management system (e.g. HashiCorp Vault, AWS SSM). 
    - **SecretStore** - This is a namespaced resource that determines *how* an external secret will be accessed from an authentication perspective.
    - **ExternalSecret** - This is a resource that declares *what* data you want to fetch from the external secrets manager.