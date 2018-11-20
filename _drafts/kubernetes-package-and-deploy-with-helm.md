---
layout: post
title:  "Package and deploy your app on Kubernetes using Helm"
categories: kubernetes helm
excerpt: <p>Use Helm as deployment manager for Kubernetes.</p>
---

To deploy an application on Kuberetes you need to create several Kubernetes objects.
We saw that each object can be created using a `yaml` spec file.

For a simple application componed of a webserver, you'll need at least 3 kubernetes objects :
* a deployment to create a pod with replicas
* a service to ensure load balancing in front of your pods
* an ingress to allow traffic from outside

But a complex application needs much more Kubernetes objects :
* configmap(s) to allow configuration override
* secret(s) to store sensitive data
* maybe custom resource definitions ?

So you have a bunch of `yaml` files you can apply on Kubernetes to deploy you app.
You may use the following command to do so :

```shell
kubectl apply -R -f <kubernetes_files_folder>
```
This may be enough for your use case.
But how do you deal to apply same deployment accross different environments ?
Also how do you manage versioning and rollback of your deployements ? 

You can develop it your way... or you can use Helm.
Helm is a package manager for Kubernetes : you can use it to release or rollback your Kubernetes app with a default or specific configuration depending on your needs.
A Helm package is called `a chart`. A chart is basically a set of Kubernetes templates you need to deploy your app. 
A Helm chart can also depends on other charts : Helm let you manage these dependencies.

## Helm setup

Helm has 2 components :
* Helm client : a binary client to launch helm commands 
* Tiller : a server to install on the kubernetes cluster in charge of serving helm requests launched from the client. Tiller communicates directly with Kubernetes Api.

Please follow the official documentation to install helm client and tiller server : https://docs.helm.sh/using_helm/#installing-helm
In this article, I will assume that we install Tiller in a specific namespace with a tiller service account admin of this namespace :

```shell
kubectl create namespace petclinic
kubectl create serviceaccount tiller --namespace petclinic
```

Then apply the following yaml (save it in a `role-tiller.yaml` file) to create tiller-role :

```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: tiller-manager
  namespace: petclinic
rules:
- apiGroups: ["", "batch", "extensions", "apps"]
  resources: ["*"]
  verbs: ["*"]
```

```shell
oc apply -f role-tiller.yaml
```

And this yaml to assign freshly created `tiller-manager` role `petclinic` service account (save this to `rolebinding-tiller.yaml` file):

```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: tiller-binding
  namespace: petclinic
subjects:
- kind: ServiceAccount
  name: tiller
  namespace: petclinic
roleRef:
  kind: Role
  name: tiller-manager
  apiGroup: rbac.authorization.k8s.io
```

```shell
oc apply -f rolebinding-tiller.yaml
```

You are now ready to install tiller in `petclinic` namespace using `tiller` service account :

```shell
helm init --service-account tiller --tiller-namespace petclinic
```

## Create a Helm chart

Once installed, we can init our first Heml Chart (~package).
For the purpose of this article, we will bootstrap a demo application using helm. 

We will use the famous spring petclinic application ;-)
You can clone it from here : https://github.com/spring-petclinic/spring-petclinic-angularjs

The petclinic application we will deploy is composed of :
* an angular client
* a springboot backend
* a mysql database

Let's create our chart :

```shell
helm create petclinic-chart
```

This command creates a `petclinic-chart` directory with the following structure :

```shell
petclinic-chart/
├── charts                  # If our chart depends on other charts
├── Chart.yaml              # Chart definition
├── templates               # Kubernetes templates that compose our application
│   ├── deployment.yaml
│   ├── _helpers.tpl
│   ├── ingress.yaml
│   ├── NOTES.txt
│   └── service.yaml
└── values.yaml             # Default configuration values of our chart
```

We are ready to deploy this sample application.
To do so :

```
helm install petclinic-chart --tiller-namespace petclinic --namespace petclinic
```