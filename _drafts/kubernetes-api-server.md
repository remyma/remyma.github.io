---
layout: post
title:  "Kubernetes api server"
tags: [kubernetes]
excerpt: <p></p>
---

On the [previous post][] I blogged about the kubernetes API and resources.
Now let's focus on the engine that deals with the api calls : the kubernetes api-server and the control loop.

## Api request lifecycle

Kubernetes Api request will pass through a few filters before being written in etcd.

1. Authentication and Authorization
2. mutating admission
3. Object schema validation
4. Validating Admission
5. Persisting to Etcd

## 2 steps process

All api requests are stored in etcd database.
Then controllers react to resources

## Custom api server
