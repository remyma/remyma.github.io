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

[karaf-provisioning]: https://karaf.apache.org/manual/latest/provisioning

## How to create your custom karaf distribution

{% highlight shell %}
mvn archetype:generate \
  -DarchetypeGroupId=org.apache.karaf.archetypes \
  -DarchetypeArtifactId=karaf-assembly-archetype \
  -DarchetypeVersion=4.0.7 \
  -DgroupId=nl.theguild \
  -DartifactId=karaf-distro \
  -Dversion=1.0-SNAPSHOT \
  -Dpackage=nl.theguild.karaf-docker
{% endhighlight %}

## How to dockerize your karaf distribution

## How to scale your karaf distribution


You’ll find this post in your `_posts` directory. Go ahead and edit it and re-build the site to see your changes. You can rebuild the site in many different ways, but the most common way is to run `jekyll serve`, which launches a web server and auto-regenerates your site when a file is updated.

To add new posts, simply add a file in the `_posts` directory that follows the convention `YYYY-MM-DD-name-of-post.ext` and includes the necessary front matter. Take a look at the source for this post to get an idea about how it works.

Jekyll also offers powerful support for code snippets:

{% highlight ruby %}
def print_hi(name)
  puts "Hi, #{name}"
end
print_hi('Tom')
#=> prints 'Hi, Tom' to STDOUT.
{% endhighlight %}

Check out the [Jekyll docs][jekyll-docs] for more info on how to get the most out of Jekyll. File all bugs/feature requests at [Jekyll’s GitHub repo][jekyll-gh]. If you have questions, you can ask them on [Jekyll Talk][jekyll-talk].

[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
