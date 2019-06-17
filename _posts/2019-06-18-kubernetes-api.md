---
layout: post
title:  "Kubernetes api"
tags: [kubernetes]
excerpt: <p>I hope you’ll find this post helpful, if like me, you were used to run `kubectl` commands without knowing that much about kubernetes API.</p>
---

I hope you’ll find this post helpful, if like me, you were used to run `kubectl` commands without knowing that much about kubernetes API.

## Kubernetes objects

As explained on the [official product page](http://kubernetes.io) :

> Kubernetes (K8s) is an open-source system for automating deployment, scaling, and management of containerized applications.

To accomplish this, Kubernetes introduce some concepts which are modeled as objects. 
Most common kubernetes objects are :


* pod
* service
* deployment
...

All this objects are exposed as Api resources using kubernetes Api.

Using `kubectl` client, you can manage Kubernetes Api resources, for example if you want to list pods :

```shell
➜ kubectl get pods
```

A kubernetes resource can be specified in yaml or json format. Here is a yaml which represents a `deployment` :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
spec:
  ...
```

Every kubernetes object definition must contain these 2 fields : 
* apiVersion - Which version of the Kubernetes API you’re using to create this object
* kind - What kind of object you want to create

But, which are the available resources supported on your kubernetes cluster ? 
Also which api versions are supported ?

## Api resources

You can list all available Api resources along with given action verbs using this command :

```shell
➜ kubectl api-resources -o wide
```


This command will print similar output : 

```shell 
NAME                              SHORTNAMES   APIGROUP                       NAMESPACED   KIND                             VERBS
bindings                                                                      true         Binding                          [create]
componentstatuses                 cs                                          false        ComponentStatus                  [get list]
configmaps                        cm                                          true         ConfigMap                        [create delete deletecollection get list patch update watch]
endpoints                         ep                                          true         Endpoints                        [create delete deletecollection get list patch update watch]
events                            ev                                          true         Event                            [create delete deletecollection get list patch update watch]
limitranges                       limits                                      true         LimitRange                       [create delete deletecollection get list patch update watch]
namespaces                        ns                                          false        Namespace                        [create delete get list patch update watch]
nodes                             no                                          false        Node                             [create delete deletecollection get list patch update watch]
persistentvolumeclaims            pvc                                         true         PersistentVolumeClaim            [create delete deletecollection get list patch update watch]
persistentvolumes                 pv                                          false        PersistentVolume                 [create delete deletecollection get list patch update watch]
pods                              po                                          true         Pod                              [create delete deletecollection get list patch update watch]
podtemplates                                                                  true         PodTemplate                      [create delete deletecollection get list patch update watch]
replicationcontrollers            rc                                          true         ReplicationController            [create delete deletecollection get list patch update watch]
resourcequotas                    quota                                       true         ResourceQuota                    [create delete deletecollection get list patch update watch]
secrets                                                                       true         Secret                           [create delete deletecollection get list patch update watch]
serviceaccounts                   sa                                          true         ServiceAccount                   [create delete deletecollection get list patch update watch]
services                          svc                                         true         Service                          [create delete get list patch update watch]
mutatingwebhookconfigurations                  admissionregistration.k8s.io   false        MutatingWebhookConfiguration     [create delete deletecollection get list patch update watch]
validatingwebhookconfigurations                admissionregistration.k8s.io   false        ValidatingWebhookConfiguration   [create delete deletecollection get list patch update watch]
customresourcedefinitions         crd,crds     apiextensions.k8s.io           false        CustomResourceDefinition         [create delete deletecollection get list patch update watch]
apiservices                                    apiregistration.k8s.io         false        APIService                       [create delete deletecollection get list patch update watch]
controllerrevisions                            apps                           true         ControllerRevision               [create delete deletecollection get list patch update watch]
daemonsets                        ds           apps                           true         DaemonSet                        [create delete deletecollection get list patch update watch]
deployments                       deploy       apps                           true         Deployment                       [create delete deletecollection get list patch update watch]
....
```

What do we have here :
* **NAME** : name of the resource collection
* **SHORTNAMES** : a resource can have an abreviated name. 
Example, if you want to list pvc, you can use this command `kubectl get pvc` instead of `kubectl get persistentvolumeclaims`
* **APIGROUP** : resources can be grouped in api groups. Core resources such as `pods`, `nodes`, `namespaces` have no api group.
* **NAMESPACED** : scope of the resource. If `true` resources are attached to a namespace. If `false`, resources are attached to the cluster.
Example : a `node` is a cluster resource, it is not bound to a namespace. On the contrary `pod` is a namespace resource.
* **KIND** : name of the resource
* **VERBS** : all available actions

On the previous output, you may notice that some resources (pods, services, nodes...) 
have no matching group. These are core resources (pods, services, nodes, ...), they belong to core group
(also called "legacy group").

## Api groups

### Api versions

You can check supported versions for each api groups of your kubernetes cluster using this command : 

```shell
➜ kubectl api-versions 
admissionregistration.k8s.io/v1beta1
apiextensions.k8s.io/v1beta1
apiregistration.k8s.io/v1
apiregistration.k8s.io/v1beta1
apps/v1
apps/v1beta1
apps/v1beta2
authentication.k8s.io/v1
authorization.k8s.io/v1
authorization.k8s.io/v1beta1
autoscaling/v1
autoscaling/v2beta1
autoscaling/v2beta2
batch/v1
batch/v1beta1
certificates.k8s.io/v1beta1
coordination.k8s.io/v1
coordination.k8s.io/v1beta1
extensions/v1beta1
k3s.cattle.io/v1
networking.k8s.io/v1
networking.k8s.io/v1beta1
node.k8s.io/v1beta1
policy/v1beta1
rbac.authorization.k8s.io/v1
rbac.authorization.k8s.io/v1beta1
scheduling.k8s.io/v1
scheduling.k8s.io/v1beta1
storage.k8s.io/v1
storage.k8s.io/v1beta1
v1
```

As you can see in the ouput, Kubernetes supports multiple API versions. For example on the kubernetes cluster I am working on
you can see that supported versions for batch api group are :

```shell
batch/v1
batch/v1beta1
```

These versions may evolve on cluster upgrades, so pay attention to it,
otherwise you may end with these kind of error messages when creating a resource :

```shell
error: server does not support API version "batch/v2alpha1"
```

## REST Api

All kubernetes Api resources are exposed through a secured REST Http Api.

The simplest way to communicate with Kubernetes REST Api without having to deal with authentication and complex url
is to start a proxy server on your local machine, so that you can access to the cluster url as `localhost`.

Running a kubernetes proxy is as simple as this :

```shell
➜ kubectl proxy                    
Starting to serve on 127.0.0.1:8001
```

You can then browse your http api using `curl` or any other http client :

```shell
➜  ~ curl http://localhost:8001    
{
  "paths": [
    "/api",
    "/api/v1",
    ...
  ]
}
```

you can browse the api this way : 

`http://localhost:8001/apis/$GROUP_NAME/$VERSION/$RESOURCE_NAME`

For example, if you want to get pods in the namespace `default` :

```shell
curl http://localhost:8001/api/v1/namespaces/default/pods
```

If you want to discover the api in details, there is an [openApi endpoint](https://kubernetes.io/docs/concepts/overview/kubernetes-api/#openapi-and-swagger-definitions) :
[http://127.0.0.1:8001/openapi/v2](http://127.0.0.1:8001/openapi/v2)

Load this openApi spec with an api tool like postman or insomnia, and you'll be able to use it easily :

![Kubernetes openApi]({{ "/assets/images/posts/kubernetes-api/swagger.png" | absolute_url }})

## Kubernetes api clients

As we have seen, we can manage kubernetes objects either directly using http calls, or with `kubectl` command line.
But there are also [official client librairies](https://kubernetes.io/docs/reference/using-api/client-libraries/)
if you want to call kubernetes api with your favorite language.

## Conclusion

All Kubernetes resources are manageable through the Kubernetes Api. Resources are grouped with ApiGroups
and are versioned.
Simplest way to make api calls is to use `kubectl` command line, but you can also query the api using raw http calls.
If you want to call the api from your code, you can also use a client library for the language of your choice. 
