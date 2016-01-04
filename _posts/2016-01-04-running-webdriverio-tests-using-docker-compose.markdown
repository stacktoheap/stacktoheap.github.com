---
layout: post
title: "Running WebDriverIO tests using docker-compose"
excerpt: "Running WebDriverIO tests using docker-compose for development and CI"
reading_time: "7 mins"
date: 2016-01-04 23:48
comments: true
categories: [docker, docker-compose]
tags: [docker, docker-compose, webdriverio, selenium, node.js]
---

I have been in the process of setting up a Docker and docker-compose based workflow for development and CI. As part of this setup, we have been able to run automated functional tests on docker as well.

#WebDriverIO setup

Apart from the standard bootstrap steps outlined in the WebDriverIO docs, I use the awesome [`node-config`](https://github.com/lorenwest/node-config) library for configuration. This helps in neatly configuring the differences between different environments - in this case, local development setup and docker based CI.

The `test` folder has the following structure:

{% highlight bash %}
$ tree ./test/
./test/
├── config
│   ├── default.json
│   └── docker.json
├── specs
│   └── home.js
└── wdio.conf.js
{% endhighlight %}

The config files have the default driver settings in `default.json`. These are used for running the tests in development. `docker.json`, similarly, has the settings needed for docker based functional tests (which run in CI, but can also be run in development, if needed.)

For local development, we use local chrome driver for the tests. Accordingly, the default.json looks something like the following:

{% highlight json %}
{
  "driver_host": "127.0.0.1",
  "driver_port": 9515,
  "driver_path": "/",
  "local_chrome_driver": true,
  "app_base_path": "http://localhost:9090"
}
{% endhighlight %}

For docker based tests, we use standalone selenium, running in a docker container (details in the following section.) Accordingly, `docker.json` looks like the following:

{% highlight json %}
{
  "driver_host": "chrome_standalone",
  "driver_port": 4444,
  "driver_path": "/wd/hub",
  "local_chrome_driver": false,
  "app_base_path": "http://app_built:9090"
}
{% endhighlight %}

Note that `chrome_standalone` and `app_built` are hostnames populated by the link mechanisms of docker and docker-compose.

A simple npm script command is added in `package.json` for running the functional tests:

{% highlight json %}
"scripts": {
  ...
  "ft": "NODE_CONFIG_DIR=./test/config wdio ./test/wdio.conf.js"
}
{% endhighlight %}

The above allows us to run the functional tests as `npm run ft`. Docker based functional tests can be run with `NODE_ENV=docker npm run ft`.

Note: The environment variable `NODE_CONFIG_DIR` is set to `./test/config` so that tests can have their own isolated configs, and not be interleaved with configs used by the application.

#Compose file

The functional testing specific docker-compose file - called `docker-compose.fy.yml` has, in its simplest form, the following:

{% highlight yaml %}
app_built:
  image: registry/repository/app:${VERSION}
  ports:
    - "9090:9090"
  environment:
    - NODE_ENV="production"

chrome_standalone:
  image: selenium/standalone-chrome:2.48.2
  volumes:
  - /dev/shm:/dev/shm
  ports:
  - 4444:4444
  links:
  - app_built

app_ft:
  build: .
  dockerfile: Dockerfile.build
  command: bash -c "npm install && npm run ft"
  volumes:
  - .:/app:rw
  links:
  - chrome_standalone
  - app_built
  environment:
  - NODE_ENV=docker
{% endhighlight %}

The entire thing is a composition of three containers, as described below:

- *app_built* - This docker-compose service represents the actual application to be tested. The container has been built and pushed to a private registry as part of a separate build process.

- *chrome_standalone* - This is the service that runs Selenium standalone along with Chrome. More details on the `selenium/standalone-chrome` can be obtained here - [https://github.com/SeleniumHQ/docker-selenium](https://github.com/SeleniumHQ/docker-selenium). For a more mature setup, this can be replaced with a multi-container Selenium hub, also detailed in the link above. This service links to *app_built* since it needs to access the app to run the tests.

- *app_ft* - this is the service *from* which we run the functional tests. `Dockerfile.build` helps in creating a simple node.js based container that can run node.js script/commands. As described in the previous section, we also pass the `NODE_ENV` environment variable, set to `docker`. This makes the tests pick up the settings from `docker.json`

With the above compose file, we can run the functional tests using:

docker-compose -f docker-compose.ft.yml run --rm app_ft

Happy testing!
