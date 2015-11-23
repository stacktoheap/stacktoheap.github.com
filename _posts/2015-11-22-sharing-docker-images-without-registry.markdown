---
layout: post
title: "Sharing docker images without a registry"
excerpt: "Sharing docker images between developer workstations without a registry"
date: 2015-11-22 23:30
comments: true
categories: [docker]
tags: [docker, docker-machine, docker-sync, registry, docker-save, docker-load]
---

One of the well-known ways of sharing docker images is via a registry - be it the hosted ones like [DockerHub](https://hub.docker.com/) or [Quay.io](https://quay.io/), or private registries setup within your organization.

Also, once a workstation has a set of images, pulling in new versions or similar images becomes a breeze.

But we wanted to share images between developer workstations, without having to push and pull from a registry.

### Why no registry?

Recently, when we were switching to a docker based workflow, we found that pulling in the base set of images from our internal registry on AWS + S3 was taking a lot of time. Not to mention that it was also eating up our bandwidth. In order to make it easy and fast for new developers to get started with our project, we wanted to setup a local "store" of images that we can pull our images from. This store will also help us share new images quicker as well.

Instead of setting up a local registry in our office LAN, we decided to keep things simple and setup a simple workstation to pull in images via SSH. The `docker save` and `docker load` commands helped us beautifully here. We also piped from/to these command directly to/from the workstation, and avoided local files altogether.

### docker save and docker load

First, some basics about `docker save` and `docker load`:

`docker save` allows use to save a image to a tar archive. It streams to STDOUT by default, or to a file specified by the `-o, --output` flag.

`docker load` sits in the other end, and takes in input from a file, specified by the `-i, --input` flag or STDIN.

### Sync to/from external host via ssh

Once we had the workstation in our LAN (let's say its IP is 192.168.0.42), syncing an image to it is as simple as:

{% highlight bash %}
docker save internal-registry/image | bzip2 | \
  pv | ssh 192.168.0.42 "cat > ~/docker_images/image"
{% endhighlight %}

(this push can be done from a one workstation where the image was built, or pulled in from a registry)

We used `bzip2` to compress the data, and the amazing `pv` tool to visualize the transfer.

Pulling in an image is not much different:

{% highlight bash %}
ssh 192.168.0.42 "cat ~/docker_images/image" | pv | \
  bunzip2 | docker load
{% endhighlight %}

### Sync between multiple docker machines

We use [docker-machine](https://docs.docker.com/machine/) on OS X for development. This strategy can also be used to share or backup images between different docker-machines. If there are two machines, say `dev1` and `dev2` running, we can sync an image between the two like below:

{% highlight bash %}
docker $(docker-machine config dev1) save image | \
  docker $(docker-machine config dev2) load
{% endhighlight %}

If a machine (say `dev`) is going to be recreated, we can save the image(s) and reload:

{% highlight bash %}
docker $(docker-machine config dev) save -o image.tar image
docker-machine rm dev
docker-machine create --provider virtualbox dev
docker $(docker-machine config dev) load -i image.tar
{% endhighlight %}

This should make using multiple docker-machines or recreating existing ones less of an hassle in terms of getting the base set of images.

This simple setup makes pulling in images much faster, and also saves bandwidth. Hope it is useful in your work environment too.
