---
layout: post
title:  "Local jekyll in docker and with docker-compose"
date:   2017-10-02 
categories: jekyll docker docker-compose
---

`Jekyll` is a cool piece of software but its configuration on e local device may be painful for people who are not familiar with Ruby stack. To simplify this process we may use `jekyll` inside [docker](https://www.docker.com/) image. 
In this tutorial I will use official `jekyll` docker image from [docker hub](https://hub.docker.com/r/jekyll/jekyll/).
 Source code and documentation of this image is available on [github](https://github.com/jekyll/docker/blob/master/README.md).

### Let's start
In jekyll docker image documentation we may find such an example of use: 

{%  highlight bash %}
export JEKYLL_VERSION=3.5
docker run --rm \
  --volume=$PWD:/srv/jekyll \
  -it jekyll/builder:$JEKYLL_VERSION \
  jekyll build
{% endhighlight %}

The example above works fine, but the command is quite complex and hard to remember.
To simplify we may use [docker-compose](https://docs.docker.com/compose/) tool.

First we need to create `docker-compose.yml` file with content bellow 

{% highlight yml %}
services:
  site:
    image: jekyll/jekyll:latest
    environment:
      - JEKYLL_VERSION=3.5
    network_mode: "host"
    volumes:
      - $PWD:/srv/jekyll

{% endhighlight %}

And now we may build our project by typing 

{%  highlight bash %}
 $  docker-compose run site jekyll build
{% endhighlight %}

or serve

{%  highlight bash %}
 $  docker-compose run site jekyll serve
{% endhighlight %}

This approach works fine but is slow. Everything because `docker-compose` creates new container on each `run` command.
To solve this we need to add separate service definition in `docker-compose.yml` file, for each `jekyll` command that we want to execute.

{% highlight yaml %}
version: '2'
services:
  serve:
    environment:
      - JEKYLL_VERSION=3.5
    command: jekyll serve
    image: jekyll/jekyll:latest
    network_mode: "host"
    volumes:
      - $PWD:/srv/jekyll

  build:
    environment:
      - JEKYLL_VERSION=3.5
    command: jekyll build
    image: jekyll/jekyll:latest
    network_mode: "host"
    volumes:
      - $PWD:/srv/jekyll
{% endhighlight %}

And now we may build our project by typing 

{%  highlight bash %}
 $  docker-compose up build
{% endhighlight %}

or serve

{%  highlight bash %}
 $  docker-compose up serve
{% endhighlight %}


On the first run `docker-compose` will build container from scratch and execution time will be similar to the previous approach. 

{% highlight bash %}
$ docker-compose up build

Creating dev_build_1 ... 
Creating dev_build_1 ... done
Attaching to dev_build_1
build_1  | The following gems are missing
build_1  |  * jekyll-archives (2.1.1)
build_1  |  * jekyll-whiteglass (1.3.0)
build_1  | Install missing gems with `bundle install`
build_1  | Fetching gem metadata from https://rubygems.org/...........
build_1  | Fetching version metadata from https://rubygems.org/..
build_1  | Fetching dependency metadata from https://rubygems.org/.
build_1  | Using public_suffix 3.0.0
build_1  | Using bundler 1.15.4
build_1  | Using colorator 1.1.0
build_1  | Using ffi 1.9.18
build_1  | Using forwardable-extended 2.6.0
build_1  | Using rb-fsevent 0.10.2
build_1  | Using kramdown 1.15.0
build_1  | Using liquid 4.0.0
build_1  | Using mercenary 0.3.6
build_1  | Using rouge 2.2.1
build_1  | Using safe_yaml 1.0.4
build_1  | Using jekyll-paginate 1.1.0
build_1  | Using addressable 2.5.2
build_1  | Using rb-inotify 0.9.10
build_1  | Using pathutil 0.14.0
build_1  | Using sass-listen 4.0.0
build_1  | Using listen 3.0.8
build_1  | Using sass 3.5.1
build_1  | Using jekyll-watch 1.5.0
build_1  | Using jekyll-sass-converter 1.5.0
build_1  | Using jekyll 3.6.0
build_1  | Fetching jekyll-archives 2.1.1
build_1  | Installing jekyll-archives 2.1.1
build_1  | Using jekyll-feed 0.9.2
build_1  | Using jekyll-sitemap 1.1.1
build_1  | Using minima 2.1.1
build_1  | Fetching jekyll-whiteglass 1.3.0
build_1  | Installing jekyll-whiteglass 1.3.0
build_1  | Bundle complete! 5 Gemfile dependencies, 26 gems now installed.
build_1  | Bundled gems are installed into /usr/local/bundle.
build_1  | Configuration file: /srv/jekyll/_config.yml
build_1  |             Source: /srv/jekyll
build_1  |        Destination: /srv/jekyll/_site
build_1  |  Incremental build: disabled. Enable with --incremental
build_1  |       Generating... 
build_1  |                     done in 0.276 seconds.
build_1  |  Auto-regeneration: disabled. Use --watch to enable.
build_1  | Configuration file: /srv/jekyll/_config.yml
build_1  |             Source: /srv/jekyll
build_1  |        Destination: /srv/jekyll/_site
build_1  |  Incremental build: disabled. Enable with --incremental
build_1  |       Generating... 
build_1  |                     done in 0.269 seconds.
build_1  |  Auto-regeneration: disabled. Use --watch to enable.
dev_build_1 exited with code 0
{% endhighlight %}

But with the second run `docker-compose` will reuse existing container and only execute `jekyll build` without installing `gems`

{% highlight bash %}
$ docker-compose up build

Starting dev_build_1 ... 
Starting dev_build_1 ... done
Attaching to dev_build_1
build_1  | The Gemfiles dependencies are satisfied
build_1  | Configuration file: /srv/jekyll/_config.yml
build_1  |             Source: /srv/jekyll
build_1  |        Destination: /srv/jekyll/_site
build_1  |  Incremental build: disabled. Enable with --incremental
build_1  |       Generating... 
build_1  |                     done in 0.283 seconds.
build_1  |  Auto-regeneration: disabled. Use --watch to enable.
build_1  | Configuration file: /srv/jekyll/_config.yml
build_1  |             Source: /srv/jekyll
build_1  |        Destination: /srv/jekyll/_site
build_1  |  Incremental build: disabled. Enable with --incremental
build_1  |       Generating... 
build_1  |                     done in 0.288 seconds.
build_1  |  Auto-regeneration: disabled. Use --watch to enable.
dev_build_1 exited with code 0

{% endhighlight %}

We could stop off at this point but we can do it even better. Currently each service definition contains some recurring configuration options like:

{% highlight yaml %}
    environment:
      - JEKYLL_VERSION=3.5
    image: jekyll/jekyll:latest
    network_mode: "host"
    volumes:
      - ${PWD}:/srv/jekylls
{% endhighlight %}

We may resolve this using [extends](https://docs.docker.com/compose/extends/) field.
To do it we need to split our configuration into two separate files. The first will contain recurring definition, the second will provide custom configuration fields.  

 `docker-compose-base.yml`
{% highlight yaml %}
version: '2'
services:
  jekyll:
    environment:
      - JEKYLL_VERSION=3.5
    image: jekyll/jekyll:latest
    network_mode: "host"
    volumes:
      - ${PWD}:/srv/jekylls
{% endhighlight %}


 `docker-compose.yml`
{% highlight yaml %}
version: '2'
services:
  serve:
    extends: 
      file: docker-compose-base.yml
      service: jekyll 
    command: jekyll serve
  
  build:
    extends: 
      file: docker-compose-base.yml
      service: jekyll 
    command: jekyll build 
{% endhighlight %}

This modification has no impact on how we start `jekyll` commands. Everything is the same as in the previous example.

{%  highlight bash %}
 $  docker-compose up build
 $  docker-compose up serve
{% endhighlight %}