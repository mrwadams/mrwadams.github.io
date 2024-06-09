---
layout: post
title: "Running Red Hat OpenShift Local on an M3 Macbook Pro"
date: 2024-06-08 00:00:00 -0000
categories:
tags: [openshift, kubernetes, macbook]
---

## Introduction

Red Hat OpenShift Local is a Kubernetes distribution that is optimized for developer desktops. It provides a consistent environment for developing, testing, and deploying applications on Kubernetes. In this post, I'll be documenting how to run Red Hat OpenShift Local on an M3 Macbook Pro, along with some tips for overcoming common issues.

## Prerequisites
Sign up for a Red Hat Developer account [here](https://developers.redhat.com/) in order to access the download link for Red Hat OpenShift Local.

## Installation
1. In the Red Hat Hybrid Cloud Console, go to Clusters -> [Local](https://console.redhat.com/openshift/create/local)
2. Select `MacOS` and `aarch64` in the dropdown fields, then click `Download OpenShift Local`.
3. Don't forget to download or copy the `pull-secret.txt` file as you'll need this during the installation process.
4. Once the download is complete, run the package installer and follow the on-screen instructions.
5. After the packagage install is complete, run `crc setup` to run the full installation process.
6. After the installation is complete, run `crc start` to start the OpenShift Local cluster. You'll be prompted to enter the `pull-secret.txt` file you downloaded earlier.
7. Once the cluster is up and running, you can access the OpenShift console by navigating to the URL provided in the terminal and logging in with the credentials provided.
8. You should now see something similar to the screenshot below.

![OpenShift Console](/assets/images/openshift-console.png)

## References

[Red Hat OpenShift Local - Overview](https://developers.redhat.com/products/openshift-local/overview)