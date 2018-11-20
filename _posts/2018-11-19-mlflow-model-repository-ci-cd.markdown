---
layout: post
title: "MLFlow as a model repository in your CI/CD workflow"
excerpt: "We recently started using MLFlow as a model repository and integrated it with GoCD"
reading_time: "4 mins"
date: 2018-11-19 10:45
comments: true
categories: [ml]
tags: [ml, mlflow, gocd, machine-learning]
image: "mlflow-vsm.png"
---

[MLFlow](https://mlflow.org/) is an open source platform for the entire end-to-end machine learning lifecycle. At Indix, we saw it as a good fit within our interal ML Platform as a model repository. We also integrated MLFlow with GoCD, our CI/CD tool of choice for years.

## Requirements from a model repository

We see a model repository as being similar to other artifact repositories like Maven and Ivy. It should help us to add and track models based on different libraries (scikit-learn, MLLib, fastText etc.) along with all the associated metadata like the hyperparams and metrics. Essentially, everything that went into training the model (the notebook itself or library version, training data, hyper params etc) and all the output (including the model themselves alongwith all the relevant metrics) should be versioned and available for consumption.

## MLFlow as a model repository.

MLFlow is a complete end-to-end machine learning lifecycle platform. The missing piece in our internal ML Platform has been the model repository and MLFlow fit in pretty well. For this purpose we utilize the MLFlow Tracking API and the UI to track our `experiments` and the different `runs` within them as we iterate on the models.

![mlflow-ui]({{ site.url }}/images/mlflow-ui.png)

## MLFlow integration with GoCD

Our CI/CD process involves taking builds out of promoted models and performing test set validations, delta stats from previous version, containerization, deployments for online predictions and creating AWS AMIs backed with the model containers for offline batch processing.

At every stage, it is essential that it is possible to track back to the model version being used and even the training data and hyperparams used to build the said model. MLFLow makes it easy to track all the inputs and outputs of each and every model building process. GoCD, the open source CI/CD tool from ThoughtWorks makes it trivial to track artifacts as they flow through various CD pipelines. It was a no-brainer that we ended up integrating MLFlow as a package repository in GoCD so that a model deployed in production can be traced back to its corresponding run all the way back to MLFlow.

![mlflow-vsm]({{ site.url }}/images/mlflow-vsm.png)

A run within an experiment that is deemed to be production ready is tagged with a "promote" tag and any time a model is promoted, a new build is triggered on GoCD just like how a new commit to Git triggers a new build.

More details about the [open source mlflow-gocd plugin can be found here](https://github.com/indix/mlflow-gocd)

## Conclusion

MLFlow is an amazing and evolving ML lifecycle tool. It can also be used in parts within your already existing tools and platform. MLFlow with GoCD is the right fit for us at Indix for our level of maturity in building and managing our own ML platforms. Watch this space for more evolutions on this.


