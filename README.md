### Introduction

This is an OpenShift demo showing how to do GitOps in a kubernetes way using tools like [ArgoCD](https://argoproj.github.io/argo-cd/) and [Kustomize](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/).

Please view the main [product-catalog](https://github.com/gnunn-gitops/product-catalog) repository for information about this demo.

This repository contains the files needed to install the demo in your own cluster. It heavily leverages kustomize's ability to support remote repositories and provides an overlay template you can use to install the application.

### Pre-Requisites

The demo does require a number of pre-requisites to work including:

a. An OpenShift 4.3 cluster or later
b. An installation of ArgoCD in that cluster
c. [OpenShift Pipelines](https://www.openshift.com/learn/topics/pipelines) operator be installed
d. [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets) operator be installed
e. An external image registry where you have rights to pull and push images, the demo is configured for [quay.io](https://quay.io/repository) but you can use anything. Note the demo assumes that the repos are public, so a credential is needed to push images but not pull.

##### ArgoCD

If you are not familiar with ArgoCD, the Canadian Solution Architecture team provides an [argocd](https://github.com/redhat-canada-gitops/argocd) repository that makes it easy to install with a reasonable ot-of-the-box configuration.

##### OpenShift Pipelines

You can install OpenShift Pipelines though the Console GUI or via Kustomize and ArgoCD. If you want to do the latter you can find a sample kustomize for it [here](https://github.com/redhat-canada-gitops/catalog/tree/master/pipelines-operator/base)

##### Sealed Secrets

One of the challenges with GitOps is handling secrets since the secrets need to be in a git repository. While it is possible to have your secrets provided from a private repo this isn't an option in many cases. Additionally even with a private repo more levels of security are highly desirables.

Sealed Secrets provides an operator where you specify a CR called a SealedSecret which has an encrypted version of the secret. When deployed to the cluster the controller will automatically decrypt it back into a plain secret. The SealedSecret is what is stored in the git repository, never the secret itself.

It's beyond the scope of this document to go into depth on Sealed Secrets however do be aware that the key used by Sealed Secrets is different for any cluster. If your cluster is ephemeral you will likely want to backup and restore the key so that you do not have to constantly update the SealedSecret in the git repo.

The Canadian SA team provides a kustomize and ArgoCD application for Sealed Secrets along with some other standard cluster configuration items [here](https://github.com/redhat-canada-gitops/cluster-config).

Other mechanisms could be used here as well (SOPS, Vault, etc) but in our team we've standardized on Sealed Secrets.

### Installation

1. Ensure

2. Fork this repo into your space

3. Edit the template overlay to match your cluster
