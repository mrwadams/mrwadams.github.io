---
layout: post
title: "CIS benchmarking Kubernetes clusters with kube-bench"
date: 2022-12-16 00:00:00 -0000
categories:
tags: [kube-bench,CIS]
---

## Background
The purpose of this post is to document the key steps required to install and run [kube-bench](https://aquasecurity.github.io/kube-bench) on a Kubernetes cluster.

## Install
Download and install the kube-bench binary on to each node in the cluster (i.e. masters and workers).

:memo: **Note:** On managed Kubernetes clusters (e.g. EKS, AKS, etc.) you will not have access to the master nodes, but can still run scans against worker nodes.

```bash
curl -L https://github.com/aquasecurity/kube-bench/releases/download/v0.6.2/kube-bench_0.6.2_linux_amd64.deb -o kube-bench_0.6.2_linux_amd64.deb

sudo apt install ./kube-bench_0.6.2_linux_amd64.deb -f
```
## Run
After the installation is complete you can run kube-bench by simply entering `kube-bench` followed by any flags you want to specify. As an example, the following command runs kube-bench and outputs the results to a file called `results.json`.

```bash
kube-bench --outputfile results.json --json
```
You can then use `jq` to display the json file in the CLI in a 'prettified' format...

```bash
jq . results.json
```

... or modify the original command to pipe the output directly to `jq`.

```bash
kube-bench --json | jq
```

## Utilising scan output
Review the output from kube-bench, paying particular attention to the recommendations to fix the identified benchmark gaps.

The recommendations are quite detailed. Often simply copying and pasting the commands they refer to will be enough to close a finding, after which you can re-run the scan and check the output to confirm.

## References
[Kube-bench Documentation](https://aquasecurity.github.io/kube-bench/v0.6.8/)
