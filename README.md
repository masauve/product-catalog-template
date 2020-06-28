### Introduction

This is an OpenShift demo showing how to do GitOps in a kubernetes way using tools like [ArgoCD](https://argoproj.github.io/argo-cd/) and [Kustomize](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/).

Please view the main [product-catalog](https://github.com/gnunn-gitops/product-catalog) repository for information about this demo.

This repository contains the files needed to install the demo in your own cluster. It heavily leverages kustomize's ability to support remote repositories and provides an overlay template you can use to install the application.

### Pre-Requisites

The demo does require a number of pre-requisites to work including:

1. An OpenShift 4.3 cluster or later
2. An installation of ArgoCD in that cluster
3. [OpenShift Pipelines](https://www.openshift.com/learn/topics/pipelines) operator be installed
4. [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets) operator be installed
5. An external image registry where you have rights to pull and push images, the demo is configured for [quay.io](https://quay.io/repository) but you can use anything. Note the demo assumes that the repos are public, so a credential is needed to push images but not pull.

##### ArgoCD

If you are not familiar with ArgoCD, the Canadian Solution Architecture team provides an [argocd](https://github.com/redhat-canada-gitops/argocd) repository that makes it easy to install with a reasonable ot-of-the-box configuration.

Note this repo currently expects argocd to be installed in the ```argocd``` namespace.

##### OpenShift Pipelines

You can install OpenShift Pipelines though the Console GUI or via Kustomize and ArgoCD. If you want to do the latter you can find a sample kustomize for it [here](https://github.com/redhat-canada-gitops/catalog/tree/master/pipelines-operator/base)

##### Sealed Secrets

One of the challenges with GitOps is handling secrets since the secrets need to be in a git repository. While it is possible to have your secrets provided from a private repo this isn't an option in many cases. Additionally even with a private repo more levels of security are highly desirables.

Sealed Secrets provides an operator where you specify a CR called a SealedSecret which has an encrypted version of the secret. When deployed to the cluster the controller will automatically decrypt it back into a plain secret. The SealedSecret is what is stored in the git repository, never the secret itself.

It's beyond the scope of this document to go into depth on Sealed Secrets however do be aware that the key used by Sealed Secrets is different for any cluster. If your cluster is ephemeral you will likely want to backup and restore the key so that you do not have to constantly update the SealedSecret in the git repo.

The Canadian SA team provides a kustomize and ArgoCD application for Sealed Secrets along with some other standard cluster configuration items [here](https://github.com/redhat-canada-gitops/cluster-config).

You are free to use other mechanisms could be used here as well (SOPS, Vault, etc) but in our team we've standardized on Sealed Secrets so this is what is shown here.

### Installation

At a high level the installation process is as follows:

1. Ensure you have the pre-requisites in place

2. Fork this repo in github

3. Copy ```clusters/overlays/template``` to ```clusters/overlays/<your-cluster-name>```

4. Edit your cluster overlay to match your cluster specifications, more details on that below.

5. Apply the argocd applications.

```oc apply -k clusters/overlays/template/tools/argocd```

With regards to point #4, here are the specific things you need to modify:

#### cicd

You will need to replace the sealed secrets for github and docker here with ones that align with your sealed secrets key and credentials for your environment.

The dockerconfig secret that you need to configure needs to have credentials for the remote registry (i.e quay.io that you want to push to).

To create the docker secret, use oc to create a regular secret in your cluster:

```
oc create secret docker-registry <pull_secret_name> \
    --docker-server=<registry_server> \
    --docker-username=<user_name> \
    --docker-password=<password> \
    --docker-email=<email>
```

And the use kubeseal to create a sealedsecret equivalent of it.

The git hub secret needs your username, github personal access token as password and email address. Note that the github secret is only needed if you want to use the pipeline which creates a PR in github to push changes to production.

The github personal access token you create mush have full rights on ```repo``` plus ```read:org``` under ```admin:org``` and ```read:user``` under ```user``` to create PRs. The github secret also requires a tekton annotation as well, here is an example:

```
apiVersion: v1
kind: Secret
metadata:
  name: github
  annotations:
    tekton.dev/git-0: https://github.com
type: kubernetes.io/basic-auth
stringData:
  username: gnunn1
  password: xxxxxxxxxxxxxxxxxxxxxxxxxxxxx
  email: gnunn@somewhere.com
```

Finally there is a default rolebinding to make the project available to user1, you can remove this if not needed or change it to a different username that is appropriate for your cluster.

#### dev, test and prod

Update the patch.yaml file to reflect the wildcard address of your cluster as well as your external registry where the images will be stored.

In the prod you will also need to update the ```patch-client-deployment.yaml``` and ```patch-server-deployment.yaml``` files.

Finally similarly, like the cicd overlay you can remove or patch the user1 rolebinding as desired.

#### metrics

No changes as there are no cluster specific patches required

#### argocd

ArgoCD works by deploying ```AppProject``` and ```Application``` objects that define the applications that we want to deploy. In our case we will be deploying several applications via ArgoCD. An ArgoCD ```Application``` contains a reference to a git repo which defines what we want to deploy. In order to specify our kustomized cluster overlay we need to patch these application references to point to the git repo and path that represents our cluster.

Therefore in each of the ```patch-xxxx-app.yaml``` files, modify the patches to point to your forked repo and the path in the repo containing the kustomized settings for that application. The changes that need to be made have been parameterized so it should be pretty obvious what needs to be modified, here is an example:

```
- op: replace
  path: /spec/source/repoURL
  value: https://github.com/<Your-org>/product-catalog-template
- op: replace
  path: /spec/source/path
  value: clusters/overlays/<your-cluster>/test
```

In the above ```<Your-org>``` is where you forked this repo, ```<your-cluster>``` is what you copied the ```template``` folder too.