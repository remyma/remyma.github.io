---
layout: post
title:  "Dockerize your own karaf distribution"
date:   2017-12-12 12:05:11 +0100
categories: karaf docker
---

Karaf is a powerful osgi runtime environment which you can use to run your java apps: from web applications to integration services.
In this post I'll explain how you can use Karaf with docker to make easier your developments and deployments.

* Toc
{:toc}

## Karaf provisioning

By karaf provisioning, we means : how to deploy and run your custom code in karaf ?

There are several ways to deploy your code in karaf. It is described [here in the official documentation][karaf-provisioning].

To sum up, the recommanded way to provision a karaf server is to package your application code as osgi features, then to install these features using karaf shell commands.

So the workflow would be something like this :
1. Download karaf server from [Download karaf server][karaf-download]
2. Install the karaf server and starts it by lauching `karaf/bin/start` command
3. Connect to the karaf shell by launching `karaf/bin/client`
4. Install the features repositories needed (ie. where to find the features)
5. Install the features you need

For instance, if I want to install `camel-core` in my karaf server, I will use these two commands from karaf shell :

{% highlight shell %}
karaf@root()> feature:repo-add mvn:org.apache.camel.karaf/apache-camel/2.19.4/xml/features
karaf@root()> feature:install camel-core
{% endhighlight %}

According to me this method has a huge drawback in a container world. Let's say you have a container to manage your karaf server, each time you will have to install a feature in it, you will modify your container by applying configuration in it, breaking the rule of idempotence a container should follow. 

To respect idempotence, it would be better to package your karaf server with all features you need, then build it and run it as container.

Fortunatly, karaf offers a way to package [your own karaf server distribution][karaf-custom-distribution]


## How to create your custom karaf distribution

Prerequisites :
* You need to have a valid maven installation to run following examples
* Current version of karaf at the time of writing this article is `4.1.4`.


You can generate your own karaf distribution by using the maven archetype : `karaf-assembly-archetype`.
For the purpose of this article, we will init a `karaf-sandbox` project. 
The `karaf-sandbox` project will consists of a karaf standard installation packaged with a [hawtio web console][hawtio].


Launch the following command from this directory to init the karaf distribution (Change groupId, package, ... according to your needs).

{% highlight shell %}
mvn archetype:generate \
  -DarchetypeGroupId=org.apache.karaf.archetypes \
  -DarchetypeArtifactId=karaf-assembly-archetype \
  -DarchetypeVersion=4.1.4 \
  -DgroupId=marem.karaf.sandbox \
  -DartifactId=karaf-sandbox \
  -Dversion=1.0.0 \
  -Dpackage=marem.karaf.sandbox
{% endhighlight %}

This command will generate a directory `karaf-sandbox` with a well formatted `pom.xml` in it.

In the generated `pom.xml` you'll find the following insteresting parts :

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

1. packaging : `karaf-assembly`
2. dependencies : all the dependencies listed here will be included in the generated package.
3. karaf-maven-plugin : you can configure the features you want to install

Then, if you launch a maven install for this project, this will package a karaf server in your `target` directory. You'll find here :
* `assembly` directory : your karaf server
* `tar.gz` : your karaf server packaged as tar
* `zip` : your karaf server packae as zip

You will be able to start your karaf server by launching `bin/karaf` command directly from the `target/assembly` directory (provided that you have a valid installation whith `JAVA_HOME` env variable parametred).

{% highlight shell %}
> mvn clean install
...

> ll target 
total 67M
drwxrwxr-x 8 4,0K déc.  27 15:12 assembly
-rw-rw-r-- 1 34M  déc.  27 15:12 karaf-sandbox-1.0.0.tar.gz
-rw-rw-r-- 1 34M  déc.  27 15:12 karaf-sandbox-1.0.0.zip

>./target/assembly/bin/karaf
{% endhighlight %}

To add `hawtio` feature to your karaf distribution, you will need to add the `hawtio` xml features dependency in the maven dependencies, then add the feature `hawtio` (or `hawtio-offline`, the one I use in this article) to the boot features, so that I will be installed during karaf boot.

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
            <installedFeatures>
                <feature>wrapper</feature>
            </installedFeatures>
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

No if you install re-install your project with maven, and launch it, hawtio will be installed and you will be able to connect to hawtio web interface : [http://localhost:8181/hawtio](http://localhost:8181/hawtio){:target="_blank"}.

Same could be done with your custom application code. Just package your application code as maven features, then add the repository xml feature in dependencies section, and finally add your custom features to bootFeatures list.

## How to dockerize your karaf distribution

## How to scale your karaf distribution

Check out the [Jekyll docs][jekyll-docs] for more info on how to get the most out of Jekyll. File all bugs/feature requests at [Jekyll’s GitHub repo][jekyll-gh]. If you have questions, you can ask them on [Jekyll Talk][jekyll-talk].

[karaf-download]: [http://karaf.apache.org/download.html]
[karaf-provisioning]: https://karaf.apache.org/manual/latest/provisioning
[karaf-custom-distribution]:https://github.com/apache/karaf/blob/master/manual/src/main/asciidoc/developer-guide/custom-distribution.adoc
[hawtio]: http://hawt.io/

[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
