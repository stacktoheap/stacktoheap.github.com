---
layout: post
title: "Hosting an S3 backed Docker Registry on Kubernetes"
excerpt: "Sharing docker images between developer workstations without a registry"
reading_time: "6 mins"
date: 2016-01-01 23:20
comments: true
categories: [docker, kubernetes]
tags: [docker, kubernetes, docker-registry, S3]
---

Running a docker registry (v2) on Kubernetes is [well documented as an addon to Kubernetes](https://github.com/kubernetes/kubernetes/tree/master/cluster/addons/registry).

That setup, however, involves proxying the registry as `localhost` on each Kubernetes node. While this simplifies pulling on nodes (no insecure registry issue, as it is localhost), this makes building and pushing outside the Kubernetes cluster unnecessarily complex and hacky  (you need to `kubectl port-forward` to access the registry, and you also must build your images with the tag like `localhost:5000/repository/image:version`.) Moreover, it is based on Persistent Volume storage.

For a better docker registry setup, we wanted two things:

 - S3 backed registry so that storage is managed better.
 - Proper service for registry so that push and pull are more sane, and image tags are proper. We would like to push and pull from local workstation and our CI boxes. Also, at any time we can move to a different hosting solution for our private registry without have to retag and push images.

For S3 storage, we can utilize the ability to override all the configuration for the registry via environment variables. Our `ReplicationController` looks like the following:

{% highlight yaml %}
apiVersion: v1
kind: ReplicationController
metadata:
  name: kube-registry-v0
  namespace: kube-system
  labels:
    k8s-app: kube-registry
    version: v0
    kubernetes.io/cluster-service: "true"
spec:
  replicas: 1
  selector:
    k8s-app: kube-registry
    version: v0
  template:
    metadata:
      labels:
        k8s-app: kube-registry
        version: v0
        kubernetes.io/cluster-service: "true"
    spec:
      containers:
      - name: registry
        image: registry:2.2.1
        env:
        - name: REGISTRY_HTTP_ADDR
          value: :5000
        - name: REGISTRY_STORAGE
          value: S3
        - name: REGISTRY_STORAGE_S3_ACCESSKEY
          value: <access_key>
        - name: REGISTRY_STORAGE_S3_SECRETKEY
          value: <secret_key>
        - name: REGISTRY_STORAGE_S3_REGION
          value: us-east-1
        - name: REGISTRY_STORAGE_S3_BUCKET
          value: <S3_bucket>
        - name: REGISTRY_STORAGE_S3_ENCRYPT
          value: "true"
        - name: REGISTRY_STORAGE_S3_SECURE
          value: "true"
        - name: REGISTRY_STORAGE_S3_V4AUTH
          value: "true"
        - name: REGISTRY_STORAGE_S3_CHUNKSIZE
          value: "5242880"
        - name: REGISTRY_HTTP_SECRET
          value: "<secret>"
        ports:
        - containerPort: 5000
          name: registry
          protocol: TCP
{% endhighlight %}

It is important to set `REGISTRY_STORAGE` to `S3` so that the default storage configuration is overridden. If this is not done, you will get an error regarding multiple storage drivers. `REGISTRY_HTTP_SECRET` has been added so that load balancing across multiple pods will work, when needed. Rest of the settings are pretty standard for a S3 backed registry, as per the docs.

We have a service that looks like below (For context, our Kubernetes cluster is on AWS, and has AWS aware features enabled):

{% highlight yaml %}
apiVersion: v1
kind: Service
metadata:
  name: kube-registry
  namespace: kube-system
  labels:
    k8s-app: kube-registry
    kubernetes.io/cluster-service: "true"
    kubernetes.io/name: "KubeRegistry"
spec:
  selector:
    k8s-app: kube-registry
  type: LoadBalancer
  ports:
  - name: registry
    port: 80
    targetPort: 5000
    protocol: TCP
{% endhighlight %}

We have a nice Route53 alias for the resulting ELB so that we can push and pull like we would to any other private registry. With the DNS name and S3 storage, moving away from Kubernetes for the registry is trivial too.

Improvements: Obviously, we are running an insecure registry at the moment. That's something on our TODO list of things to fix. Currently, our CoreOS nodes, local workstations and CI boxes have Docker service running with the `--insecure-registry` flag.
