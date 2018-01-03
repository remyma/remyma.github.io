---
layout: post
title:  "Dockerize your own karaf distribution"
date:   2018-01-03 12:05:11 +0100
categories: karaf docker
---

In this post I'll explain how you can use docker to facilitate your developments and deployments when working with [apache Karaf][karaf].

To help you run the examples in this article, I've created this [github repository][github-karaf-sandbox]{:target="_blank"}. It consists of a Maven multimodule project :
* karaf-sandbox : parent module. Also contains docker definitions
* karaf-sandbox-distribution : karaf custom distribution packaging
* karaf-sandbox-rest : a custom rest service that exposes /api/message http endpoint to post and get messages.

This post walks through the following sections :

* Toc
{:toc}

## [Karaf provisioning](#karaf-provisioning)

By karaf provisioning, we means : how to deploy and run your custom code in karaf ?

Karaf is a powerful osgi runtime environment which you can use to run your java apps : web applications, api, integration services...
There are several ways to deploy your code in karaf. It is described [here in the official documentation][karaf-provisioning]{:target="_blank"}.

To sum up, the recommanded way to provision a karaf server is to package your application code as osgi features, then to install these features using karaf shell commands.

So the workflow for provisioning a karaf server from scratch would be something like this :
1. Download karaf server archive from [Download karaf server][karaf-download]{:target="_blank"}
2. Unpack the karaf server and starts it by lauching `karaf/bin/start` command
3. Connect to the karaf shell by launching `karaf/bin/client`
4. Install the features repositories needed (ie. where to find the features)
5. Install the features you need

For instance, if I want to install `camel-core` in my karaf server, I will first connect to my karaf shell console, then use these two commands :

{% highlight shell %}
➜ karaf ./bin/client                                          
Logging in as karaf
        __ __                  ____      
       / //_/____ __________ _/ __/      
      / ,<  / __ `/ ___/ __ `/ /_        
     / /| |/ /_/ / /  / /_/ / __/        
    /_/ |_|\__,_/_/   \__,_/_/         

  Apache Karaf (4.1.4)

Hit '<tab>' for a list of available commands
and '[cmd] --help' for help on a specific command.
Hit 'system:shutdown' to shutdown Karaf.
Hit '<ctrl-d>' or type 'logout' to disconnect shell from current session.

karaf@root()> feature:repo-add mvn:org.apache.camel.karaf/apache-camel/2.19.4/xml/features
karaf@root()> feature:install camel-core
{% endhighlight %}

According to me this method has a huge drawback in a container world. Let's say you have a container to manage your karaf server, each time you will have to install a feature in it, you will modify your container by applying configuration in it, breaking the rule of idempotence a container should follow. 

To respect idempotence, it would be better to package your karaf server with all features you need, then build it and run it as container.

Fortunatly, karaf offers a way to package [your own karaf server distribution][karaf-custom-distribution]{:target="_blank"}

## [How to create your custom karaf distribution](#how-to-create-your-custom-karaf-distribution)

Prerequisites :
* You need to have a valid maven installation to run following examples
* Current version of karaf at the time of writing this article is `4.1.4`.


You can generate your own karaf distribution by using the maven archetype : `karaf-assembly-archetype`.
For the purpose of this article, we will init a `karaf-sandbox-ditribution` project. 
The `karaf-sandbox-ditribution` project will consists of a karaf standard installation packaged with [hawtio web console][hawtio]{:target="_blank"}.

### Init your custom distribution

Launch the following command from this directory to init the karaf distribution (Change groupId, package, ... according to your needs).

{% highlight shell %}
mvn archetype:generate \
  -DarchetypeGroupId=org.apache.karaf.archetypes \
  -DarchetypeArtifactId=karaf-assembly-archetype \
  -DarchetypeVersion=4.1.4 \
  -DgroupId=marem.karaf.sandbox \
  -DartifactId=karaf-sandbox-ditribution \
  -Dversion=1.0.0 \
  -Dpackage=marem.karaf.sandbox
{% endhighlight %}

This command will generate a directory `karaf-sandbox-ditribution` with a well formatted `pom.xml` in it.

In the generated `pom.xml` you'll find the following insteresting parts :

1. packaging : `karaf-assembly`
2. dependencies : all the dependencies listed here will be included in the generated package.
3. karaf-maven-plugin : you can configure the features you want to install

{% highlight xml %}
<project xmlns="http://maven.apache.org/POM/4.0.0" ...>
...

  <packaging>karaf-assembly</packaging>

  ...

  <dependencies>
    <dependency>
        <groupId>org.apache.karaf.features</groupId>
        <artifactId>framework</artifactId>
        <version>4.1.4</version>
        <type>kar</type>
    </dependency>
    <dependency>
        <groupId>org.apache.karaf.features</groupId>
        <artifactId>standard</artifactId>
        <version>4.1.4</version>
        <classifier>features</classifier>
        <type>xml</type>
    </dependency>
    ...
  </dependencies>
  <build>
    ...
    <plugins>
      ...
      <plugin>
        <groupId>org.apache.karaf.tooling</groupId>
        <artifactId>karaf-maven-plugin</artifactId>
        <configuration>
            <installedFeatures>
                <feature>wrapper</feature>
            </installedFeatures>
            <!-- <startupFeatures/> -->
            <bootFeatures>
                <!-- standard distribution -->
                <feature>standard</feature>
                <!-- minimal distribution -->
                <!--<feature>minimal</feature>-->
            </bootFeatures>
            <javase>1.8</javase>
        </configuration>
      </plugin>
    </plugins>
  </build>
</project>

{% endhighlight %}

Then, if you launch a maven install for this project, this will package a karaf server in your `target` directory. You'll find here :
* `assembly` directory : your karaf server
* `tar.gz` : your karaf server packaged as tar
* `zip` : your karaf server packae as zip

You will be able to start your karaf server by launching `bin/karaf` command directly from the `target/assembly` directory (provided that you have a valid installation whith `JAVA_HOME` env variable parametred).

{% highlight shell %}
> mvn clean install
...

> ll target 
total 144M
drwxrwxr-x 9 assembly
-rw-rw-r-- 1 karaf-sandbox-ditribution-1.0.0.tar.gz
-rw-rw-r-- 1 karaf-sandbox-ditribution-1.0.0.zip


>./target/assembly/bin/karaf
{% endhighlight %}

### Add your custom features

To add `hawtio` feature to your karaf distribution, you will need to add the `hawtio` xml features dependency in the maven dependencies, then add the feature `hawtio` (or `hawtio-offline`, the one I use in this article) to the boot features, so that I will be installed during karaf boot.

I also removed the `wrapper` feature from `installedFeatures` section, because I installs a lot of default features I don't need.

{% highlight xml %}
  <dependencies>
    ...
    <dependency>
        <groupId>io.hawt</groupId>
        <artifactId>hawtio-karaf</artifactId>
        <version>${hawtio.version}</version>
        <classifier>features</classifier>
        <scope>runtime</scope>
        <type>xml</type>
    </dependency>
  </dependencies>
  ...
  <build>
    ...
    <plugins>
      ...
      <plugin>
        <groupId>org.apache.karaf.tooling</groupId>
        <artifactId>karaf-maven-plugin</artifactId>
        <configuration>
            <installedFeatures/>
            <!-- <startupFeatures/> -->
            <bootFeatures>
                <!-- standard distribution -->
                <feature>standard</feature>
                <feature>hawtio-offline</feature>
            </bootFeatures>
            <javase>1.8</javase>
        </configuration>
      </plugin>
    </plugins>
  </build>
{% endhighlight %}

Now if you re-install your project with maven, and start it, hawtio will be installed and you will be able to connect to hawtio web interface : [http://localhost:8181/hawtio](http://localhost:8181/hawtio){:target="_blank"}.

Same could be done with your custom application code. Just package your application code as osgi features, then add the repository xml feature in dependencies section, and finally add your custom features to bootFeatures list.

## [How to run your karaf distribution with docker](#how-to-run-your-karaf-distribution-with-docker)

Now that we know how to package a custom karaf distribution, we can focus on how to run it with docker.

What we want :
* Docker should launch maven build (so that I don't need to have maven installed on host)
* Docker should builds an image from karaf generated tar archive
* Docker should expose karaf used ports : 
  * 1099
  * 8101 : Karaf ssh default port.
  * 44444
  * 8181 : Karaf http default port.


### Docker image

Here is the `Dockerfile` I use to build my image :

{% highlight docker %}
FROM maven:3.5-jdk-8 as KARAF-BUILD

RUN mkdir /src;
WORKDIR /src/
ADD . .
RUN mvn clean install

FROM openjdk:8-jdk
MAINTAINER Matthieu Rémy

VOLUME "/root/.m2"

ENV JAVA_HOME /usr/lib/jvm/java-8-openjdk-amd64
RUN apt-get update \
  && apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
RUN mkdir /opt/karaf;
COPY --from=KARAF-BUILD /src/karaf-sandbox-distribution/target/karaf-sandbox-distribution-1.0.0.tar.gz /opt/karaf-sandbox-distribution-1.0.0.tar.gz
RUN tar --strip-components=1 -C /opt/karaf -xzf /opt/karaf-sandbox-distribution-1.0.0.tar.gz; \
    rm /opt/karaf-sandbox-distribution-1.0.0.tar.gz
RUN mkdir -p /opt/karaf/data/log && touch /opt/karaf/data/log/karaf.log
EXPOSE 1099 8101 44444 8181
COPY karaf-sandbox-distribution/entrypoint.sh /usr/local/bin/docker-entrypoint
RUN chmod +x /usr/local/bin/docker-entrypoint
ENTRYPOINT ["docker-entrypoint"]

{% endhighlight %}

This `Dockerfile` use a docker [multi-stage build][docker-multistage]{:target="_blank"}, to provide a minimal image :
first part of the dockerfile is just used build the Karaf distribution with maven.

### Build image and run container with compose

To build and run this karaf sandbox container you can use `docker-compose` with the following `docker-compose.yml` file :

{% highlight yml %}
version: '3.2'
services:
  karaf:
    build: .
    volumes:
      - type: volume
        source: m2repo
        target: /root/.m2
    ports:
      - 1099:1099
      - 8101:8101
      - 4444:4444
      - 8181:8181
    networks:
      - sandbox
networks:
  sandbox:
volumes:
  m2repo:
{% endhighlight %}

Build your docker image with this command :
{% highlight shell %}
➜  docker-compose build
{% endhighlight %}

Run a docker container with this command :
{% highlight shell %}
➜  docker-compose up -d    
WARNING: The Docker Engine you're using is running in swarm mode.

Compose does not use swarm mode to deploy services to multiple nodes in a swarm. All containers will be scheduled on the current node.

To deploy your application across the swarm, use `docker stack deploy`.

Creating network "karafsandbox_sandbox" with the default driver
Creating volume "karafsandbox_m2repo" with default driver
Creating karafsandbox_karaf_1 ... 
Creating karafsandbox_karaf_1 ... done

{% endhighlight %}

Check that your container is running :
{% highlight shell %}
➜  docker ps
CONTAINER ID        IMAGE                COMMAND               CREATED             STATUS              PORTS                                                                                                       NAMES
d8e07197cba9        karafsandbox_karaf   "docker-entrypoint"   5 seconds ago       Up 4 seconds        0.0.0.0:1099->1099/tcp, 0.0.0.0:4444->4444/tcp, 0.0.0.0:8101->8101/tcp, 0.0.0.0:8181->8181/tcp, 44444/tcp   karafsandbox_karaf_1
{% endhighlight %}

To read karaf logs :
{% highlight shell %}
➜  docker-compose logs -f
{% endhighlight %}

Then you can access your container hawtio with this url : [http://localhost:8181/hawtio/login](http://localhost:8181/hawtio/login){:target="_blank"}

Here we are, you have a custom karaf distribution running in docker container ! To go further, I will explain you how to scale this docker service in a future article.

## References

This article has been inspired in part by the following projects or articles  :

* <https://www.theguild.nl/dockerizing-a-custom-karaf-distribution-in-5-minutes/>
* <https://github.com/ANierbeck/Karaf-Vertx>
* <http://cmoulliard.github.io/camel-rest-dsl-in-action>


[docker-multistage]: https://docs.docker.com/engine/userguide/eng-image/multistage-build/#before-multi-stage-builds
[karaf]: https://karaf.apache.org/
[karaf-download]: http://karaf.apache.org/download.html
[karaf-provisioning]: https://karaf.apache.org/manual/latest/provisioning
[karaf-custom-distribution]:https://github.com/apache/karaf/blob/master/manual/src/main/asciidoc/developer-guide/custom-distribution.adoc
[github-karaf-sandbox]: https://github.com/remyma/karaf-sandbox
[hawtio]: http://hawt.io/

