---
layout: post
title:  "Package and deploy your microservice application on Kubernetes using Helm"
categories: kubernetes helm
excerpt: <p>Use Helm as deployment manager for Kubernetes.</p>
---

## Why Helm ?

To deploy an application on Kuberetes you need to create several Kubernetes objects.
We saw on [this previous blog post][kuberetes-declarative-deploy]{:target="_blank"} that each kubernetes object can be created using a `yaml` spec file and apply
to our cluster using `kubectl apply -R -f <kubernetes_files_folder>` command.

That is quite easy for a simple application as you may have only a few kubernetes objects to create.
But for a complex application with multiple dependencies, it may be not enough cause of the following reasons :
* you may want to apply different configuration depending on the environment which you deploy your application, so you want templating
* you don't want to manage duplicate kubernetes definitions for a same component you deploy more than once (eg. a mysql database)
* you want to manage versioning and rollback of your deployment

You can build your custom kubernetes deployment system (see https://medium.com/virtuslab/helm-alternative-d6568aa9d40b), or you can use Helm ;-)
Helm is a package manager for Kubernetes : you can use it to release or rollback your Kubernetes app with a default or specific configuration depending on your needs.
With Helm you benefits of dependency management : A Helm package is called `a chart`. A chart is basically a set of Kubernetes templates you need to deploy your app. A Helm chart can also depends on other charts : Helm let you manage these dependencies.

TODO : Explain Helm chart. Explain Helm repo

Let's illustrate this by trying to deploy this microservice application on the cluster :
https://github.com/spring-petclinic/spring-petclinic-microservices

This microservice application is compouned of all these services : 
* Discovery Server
* Config Server
* AngularJS frontend (API Gateway)
* Customers, Vets and Visits Services
* Tracing Server (Zipkin)
* Admin Server (Spring Boot Admin)
* Hystrix Dashboard for Circuit Breaker pattern

If you look at the [docker-compose.yml](https://github.com/spring-petclinic/spring-petclinic-microservices/blob/master/docker-compose.yml) you'll notice that almost all the services depend on `config-server` and `discovery-server` to start (and `discovery-server` depends on `config-server` to start).
Customers, Vets and Visits Services rely on h2 database, but we will configure them to use a `mysql` database instead.

So how to package it using helm : we will create a helm chart for each service. We will manage the chart requirements so that each of these chart can require the `config-server`, `discovery-server` and maybe `mysql` charts to be deployed.

But before creating our chart, let's see how to setup Helm to make everithing work.

## Helm setup

Helm has 2 components :
* Helm client : a binary client to launch helm commands 
* Tiller : a server to install on the kubernetes cluster in charge of serving helm requests launched from the client. Tiller communicates directly with Kubernetes Api.

Please follow the official documentation to install helm client and tiller server : https://docs.helm.sh/using_helm/#installing-helm
In this article, I will assume that we install Tiller in a specific namespace with a tiller service account admin of this namespace :

```shell
➜ kubectl create namespace petclinic
➜ kubectl create serviceaccount tiller --namespace petclinic
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

You should be `cluster-admin` to be allowed to apply this role.
Here is the command to grant cluster-admin role to your user :

```shell
➜ kubectl create clusterrolebinding myname-cluster-admin-binding --clusterrole=cluster-admin --user=<my_user>
```

```shell
➜ kubectl apply -f role-tiller.yaml
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
➜ kubectl apply -f rolebinding-tiller.yaml
```

You are now ready to install tiller in `petclinic` namespace using `tiller` service account :

```shell
➜ helm init --service-account tiller --tiller-namespace petclinic
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
➜ helm create petclinic-chart
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

```shell
➜ helm install petclinic-chart --tiller-namespace petclinic --namespace petclinic
```

For now it's just the deployment of an nginx server.

It's time to customize our chart to package petclinic application.
Let's remove all default template to start from scratch :

```shell
➜ rm -rf petclinic-chart/templates/*.*
```

## Add mysql chart as chart requirement

Petclinic application will depend on a mysql database.
We will rely on an existing chart to package and deploy it.

We can search the helm catalog or a stable mysql chart release :

```shell
➜ helm search | grep mysql                          
...
stable/mysql                         	0.10.2       	5.7.14                      	Fast, reliable, scalable, and easy to use open-...
...
```

And use it as a requirement for our petclinic chart. Just create a `requirements.yaml` file with the following content in the root of your chart :

```yaml
dependencies:
- name: mysql
  version: 0.10.2
  repository: https://kubernetes-charts.storage.googleapis.com/
  tags:
    - petclinic-sdatabase
```

Then re-install :
```
➜ helm install petclinic-chart --tiller-namespace petclinic --namespace petclinic
```

You should see it deployed a mysql pod :
```
➜ kubectl get pods                                                
NAME                                                  READY     STATUS             RESTARTS   AGE
rafting-salamander-mysql-6c895959d4-n4kdl             1/1       Running            0          48m
```

## Create 1 sub-chart per service



Each service will need one or more yaml templates to be deployed. We could store all the services yaml template in the `templates/` directory of our chart.
But it will be more sustainable to split our chart into sub-chart : 1 per microservice.

```shell
➜ cd charts
➜ helm create config-server 
➜ helm create discovery-server 
➜ helm create customers-service
➜ helm create visits-service
➜ helm create vets-service
➜ helm create api-gateway
➜ helm create tracing-server
➜ helm create admin-server
➜ helm create hystrix-dashboard
```

[kuberetes-declarative-deploy]: http://matthieure.me


