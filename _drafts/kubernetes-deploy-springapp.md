---
layout: post
title:  "Deploy a java application with kubernetes"
categories: kubernetes springboot
---

**Docker** brings an easy way to package your application with everything needed to run it. With **Docker** you don't need to worry of the environment where your application will run, you just need a host with **Docker** installed to be able to run your application.

But now that you have dockerized your application and that you need to run it in production, there are lot more things you need to think about.
* You need to automate the deployment of your docker containers
* Maybe you need to be able to scale your app by running several instances of it on mutiple hosts
* You need to be able to manage your container configuration
* You need to ensure zeron downtime deployments
* You need to ensure that your app is always ready and healthy
* ...

All these needs are adressed by container orchestrators. **Kubernetes** is a container orchestrator open-sourced by google.

In this article, I will illustrate some of the capabilities of **Kubernetes** by deploying a java application on a local **Kubernetes** install.
We will use the Spring pet clinic application as example. It consists of :
* A PostgreSQL database
* A REST Api on top of the database : [https://github.com/spring-petclinic/spring-petclinic-rest.git][spring-petclinic-rest]{:target="_blank"}
* A front Angular webapp : [https://github.com/spring-petclinic/spring-petclinic-angular.git][spring-petclinic-angular]{:target="_blank"}

# Run with Docker

I have built images for each of the two github projets listed above and pushed them in dockerhub as a starting point.
* [marem/springpetclinic_api](https://hub.docker.com/r/marem/springpetclinic_api/)
* [marem/springpetclinic_front](https://hub.docker.com/r/marem/springpetclinic_front/)


Here is the `docker-compose.yaml` file I use to run my application :

{% highlight yaml %}
version: '3.2'
services:
  api:
    image: marem/springpetclinic_api
    ports:
      - 9966:9966
    networks:
      - spring-petclinic
  front:
    image: marem/springpetclinic_front
    ports: 
      - 4200:80
    networks:
      - spring-petclinic
networks:
    spring-petclinic:
{% endhighlight %}

It's a stack compound of 3 services :
* api
* front
* db

To start it, use the following command :

{% highlight shell %}
# start the containers
➜ docker-compose up -d

# check that containers are running
➜ docker ps
CONTAINER ID        IMAGE                                    COMMAND                  CREATED             STATUS              PORTS                              NAMES
f69b7088c5e5        marem/springpetclinic_front              "nginx -g 'daemon of…"   About an hour ago   Up About an hour    0.0.0.0:4200->80/tcp               springpetclinicrest_front_1
9f4dd43fae82        marem/springpetclinic_api                "java -jar spring-pe…"   2 hours ago         Up About an hour    8080/tcp, 0.0.0.0:9966->9966/tcp   springpetclinicrest_api_1

{% endhighlight %}

I can then access my application with opening the following link in my browser ;

[http://localhost:4200](http://localhost:4200)

Ok, that's cool. Let's see now how we can use kubernetes to run and manage the deployment of our stack.

# Deploy on Kubernetes

## Prerequisite : run a Local kubernetes

If you have no kubernetes cluster to run your apps, you can [install minikube][install-minikube]{:target="_blank"}.
Minikube acts as a local kubernetes cluster componed of a sigle worker node. Once minikube is installed, you can start it with the following command :

{% highlight shell %}
➜ minikube start
{% endhighlight %}

Then you can access kubernetes dashboard :
{% highlight shell %}
# Launch the dashboard in your browser
➜ minikube dashboard

# Or get the dashboard url
➜ minikube dashboard --url
{% endhighlight %}

## Deploy

You are now ready to deploy your app on your kubernetes cluster. Here is the command to run the petclinic api image :

{% highlight shell %}
kubectl run petclinicapi --image=marem/springpetclinic_api:latest --port 8080
{% endhighlight %}

Once you launch this command, you can check that a pod has been created for the petclinicapi image :

{% highlight shell %}
# List pods
kubectl get pods
➜  kubectl get pods
NAME                                   READY     STATUS    RESTARTS   AGE
...
petclinicapi-55cbc7fdcc-x94cb          1/1       Running   0          2d

# Tail the logs of given pod
kubectl logs petclinicapi-55cbc7fdcc-x94cb -f
{% endhighlight %}

Your pod is ready, but it is only reachable inside the kubernetes cluster. 
To open it to the outside world, you need to `expose` it : 

{% highlight shell %}
kubectl expose deployment/petclinicapi --type="NodePort" --port 9966
{% endhighlight %}

This command will create a service for your pod. 
TODO : definition of service.

{% highlight shell %}
kubectl get service
➜  spring-petclinic-rest kubectl get service
NAME           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
...
petclinicapi   NodePort    10.109.35.249   <none>        9966:30526/TCP   2d
{% endhighlight %}

The petclinic api is now reachable and you can check in your browser : 
[http://192.168.99.100:30526/petclinic/](http://192.168.99.100:30526/petclinic/)

[spring-petclinic-rest]: https://github.com/spring-petclinic/spring-petclinic-rest.git
[spring-petclinic-angular]: https://github.com/spring-petclinic/spring-petclinic-angular.git
[install-minikube]: https://kubernetes.io/docs/tasks/tools/install-minikube/