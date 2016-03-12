---
layout: post
title: Bower & the Asset Pipeline
category: code
description: "Using Bower inside the Rails Asset Pipeline"
---

One of my biggest gripes with Rails as of late has been the way third party
assets are treated. To upgrade a front-end dependency I have to delete the old
file and copy the newer version by hand. In 2014 this just feels a little
weird, especially when tools like Bower exist.

## Configure vendor/assets
I like to make a directory within `vendor/assets` called `bower_components`.
That makes it clear that they are managed by Bower - you shouldn't be touching
them. 

In the `vendor/assets/` directory we'll also have to make a `bower.json` file.
It'll look something like this:

{% highlight json %}
{
  "name": "The Project Name",
  "dependencies": {
    "<name>": "<version>",
    "<name>": "<version>",
    "<name>": "<version>"
  }
}
{% endhighlight %}

Running `bower install` will populate the `bower_components` directory with your
dependencies.

## Configure Rails
We still have to tell Rails and Sprockets about this directory. Open up
`application.rb` and paste in the following:

{% highlight ruby %}
config.assets.paths << Rails.root.join('vendor', 'assets', 'bower_components')
{% endhighlight %}

This will let Sprockets know to look in that directory when it tries to 
find assets referenced by manifests.

Now you can reference the bower components from within your Rails app.


{% highlight javascript %}
//= require jquery
//= require bower_component_name/file
{% endhighlight %}

{% highlight css %}
@import "grid";
@import "type";
@import "bower_component_name/file"
{% endhighlight %}
