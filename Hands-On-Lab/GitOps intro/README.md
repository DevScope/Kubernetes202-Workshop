# GitOps Intro Hands-On-Lab

Content

* [GitOps Intro Hands-On-Lab](#gitops-intro-hands-on-lab)
* [GitOps Intro](#gitops-intro)
    * [Abstract and learning objectives](#abstract-and-learning-objectives)
    * [Overview](#overview)
    * [Prerequisites](#prerequisites)
    * [Installing Flux on Kubernetes](#installing-flux-on-kubernetes)
    * [Exercises](#exercises)
        * [Exercise 1: Deploy GitOps Configurations and perform basic GitOps flow](#exercise-1-deploy-gitops-configurations-and-perform-basic-gitops-flow)
        * [Exercise 2: Deploy GitOps Configurations and perform Helm-based GitOps flow](#exercise-2-deploy-gitops-configurations-and-perform-helm-based-gitops-flow)

# GitOps Intro

## Abstract and learning objectives

This lab assumes you have:

* Docker Desktop with Kubernetes feature enabled or a remote cluster
* Kubectl
* A GitHub account
* Kubernetes Lens

## Overview

## Prerequisites

1. Fork the [Hello Arc](https://github.com/likamrat/hello_arc) demo application repository.

2. (Optional) Install the “Tab Auto Refresh” extension for your browser. This will help you to show the real-time changes on the application in an automated way.

* [Microsoft Edge](https://microsoftedge.microsoft.com/addons/detail/odiofbnciojkpogljollobmhplkhmofe)
* [Google Chrome](https://chrome.google.com/webstore/detail/tab-auto-refresh/jaioibhbkffompljnnipmpkeafhpicpd?hl=en)
* [Mozilla Firefox](https://addons.mozilla.org/en-US/firefox/addon/tab-auto-refresh/)

3. Install [Chocolatey](https://chocolatey.org/install)

    ```powershell
    Set-ExecutionPolicy Bypass -Scope Process -Force; 

    [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://chocolatey.org/install.ps1'))
    ```

4. Use Chocolatey to install Helm 3

    ```code
    choco install kubernetes-helm
    ```

5. Get the Flux command-line tool

## Installing Flux on Kubernetes

1. Bootstrap Flux using the `flux` command-line tool

    ```powershell
    flux install
    ```

    ![Media1](./media/1.png)

    Or using `kubectl`

    ```powershell
    kubectl apply -f https://github.com/fluxcd/flux2/releases/latest/download/install.yaml
    ```

    ![Media2](./media/2.png)

## Exercises

The exercises are made to use mostly the command-line. Of course, some of the resources you can create via K8S Lens. We encourage you to try using the command-line first!

### Exercise 1: Deploy GitOps Configurations and perform basic GitOps flow

1. Create a namespace named `hello-arc`

    ```powershell
    kubectl create namespace hello-arc
    ```

2. Create a folder in the current working directory named `exercise1/sources`

    ```powershell
    mkdir ./exercise1/sources
    ```

3. Create a `GitRepository` manifest pointing to your fork of `hello-arc` demo application's master branch.

    ```powershell
    flux create source git hello-arc `
        --url=https://github.com/YOUR_GITHUB_USERNAME/hello_arc `
        --branch=master `
        --interval=30s -n hello-arc `
        --export > ./exercise1/sources/hello-arc.yaml
    ```

    A file should be created in the directory you created previously `exercise1/sources`. This file represents the source of truth of your infrastructure and you will use it later for creating the application Kubernetes resources.

    ![Media3](./media/3.png)

4. Apply the configuration previously created

    ```powershell
    kubectl apply -f ./exercise1/sources/hello-arc.yaml -n hello-arc
    ```

    You can check if the manifest was correctly applied by executing

    ```powershell
    kubectl get gitrepository -n hello-arc
    ```

    ![Media4](./media/4.png)

    Or using Kubernetes Lens by going to `Custom Resource Definitions > source.toolkit.fluxcd.io > Gitrepositories`

    ![Media5](./media/5.png)

5. Create a folder in the current working directory named `exercise1/sources`

    ```powershell
    mkdir ./exercise1/configurations
    ```

6. Create a Flux Kustomization manifest for your application. This configures Flux to build and apply the manifests located in the directory specified

    ```powershell
    flux create kustomization hello-arc `
        --source=hello-arc `
        --path="./yaml" `
        --prune=true `
        --validation=client `
        --interval=30s `
        -n hello-arc `
        --export > ./exercise1/configurations/hello-arc.yaml
    ```

    > **Note: This tutorial uses `Kustomize` but the forked repository is not prepared for using it. Flux2 now only uses Kustomize but you can target vanilla manifests because the `kustomize-controller` will create the necessary files for you.** 

7. Apply the configuration previously created

    ```powershell
    kubectl apply -f ./exercise1/configurations/hello-arc.yaml -n hello-arc
    ```

    You can check if the manifest was correctly applied by executing

    ```powershell
    kubectl get kustomization -n hello-arc
    ```

    ![Media6](./media/6.png)

    Or using Kubernetes Lens by going to `Custom Resource Definitions > kustomize.toolkit.fluxcd.io > Kustomizations`

8. After applying the configuration you can check the tab `Workloads > Pods` and with the `hello-arc` namespace checked you can see that the resources were already created!

    ![Media7](./media/7.png)

### Exercise 2: Deploy GitOps configurations and perform Helm-based GitOps flow
