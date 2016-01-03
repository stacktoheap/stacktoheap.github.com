---
layout: post
title: "Basics of rolling updates with Kubernetes"
excerpt: "Basics of rolling updates with Kubernetes. Based on "
reading_time: "5 mins"
date: 2016-01-03 18:48
comments: true
categories: [kubernetes]
tags: [docker, kubernetes, kubectl, rolling-updates, versions]
---

It is very easy to get started with `Pods`, `ReplicationControllers` and `Services` in Kubernetes. Doing a seamless rolling update for your application, however, needs your replication controllers to be configured in a certain way to make it work. The aim of this post is to talk about the basics of configuring a production ready ReplicationController that is ready for rolling updates.

### Barebones Template

Following is a barebones template/example of a ReplicationController to enable rolling updates:

{% highlight yaml %}
{% raw %}
apiVersion: v1
kind: ReplicationController
metadata:
  name: my-application-{{ MY_APPLICATION_VERSION }}
  labels:
    name: my-application
    version: {{ MY_APPLICATION_VERSION }}
spec:
  replicas: 1
  selector:
    app: my-application
    version: {{ MY_APPLICATION_VERSION }}
  template:
    metadata:
      labels:
        app: my-application
        version: {{ MY_APPLICATION_VERSION }}
    spec:
      containers:
      - name: my-application
        image: my-application:{{ MY_APPLICATION_VERSION }}
        ports:
        - containerPort: 8080
{% endraw %}
{% endhighlight %}

The most important sections are the labels for the RC and pod, and the selectors. These include a `version` label, whose value is incremented (actually, all that matters is that this changes between deploys) during a rolling update. In my template, I have also tied up the `version` value with the version of the image being used in the pod, but that is not strictly necessary. The approach might have to be changed slightly if your pod contains multiple containers.

As you might have guessed from the `{% raw %}{{ MY_APPLICATION_VERSION }}{% endraw %}` syntax, the above is a jinja2 template. The `MY_APPLICATION_VERSION` variable's value is supplied at deploy time (from the deployment script, pipeline etc.) The RC template is usually saved as `my-application-rc.yml.tpl`

### Deploys

For deployment, we render template into an actual RC configuration at deploy time. I use [`envtpl`](https://github.com/andreasjansson/envtpl) for this purpose in my deploy scripts. [`j2cli`](https://github.com/kolypto/j2cli) is another alternative.

My deployment script looks something like the following (simplified to highlight the salient details):

{% highlight bash %}
MY_APPLICATION_VERSION=$VERSION envtpl < my-application-rc.yml.tpl > my-application-rc.yml
OLD_CONTROLLER=$(kubectl get rc -l name=my-application -o name)
OLD_CONTROLLER=${OLD_CONTROLLER##*/}
kubectl rolling-update $OLD_CONTROLLER -f my-application-rc.yml
{% endhighlight %}

First, we render the template rc configuration into a proper yaml file. Then, we get the name of the existing controller for the application using `kubectl get` with the label selector (`-l`) option. Using this old controller name, we start our rolling deploy.

Now, your application is ready for proper rolling deploys.
