---
layout: post
title: Collections and to_partial_path
---
Rendering collections of Active Record models has become very neat with some
lovely rails shorthand. Calling

{% highlight ruby %}
<%= render @article.posts %>
{% endhighlight %}

Effectively calls:
{% highlight ruby %}
<%= render partial: "post/post", locals: { post: post } %>
{% endhighlight %}

Internally, the `render` method calls `to_partial_path` on each item of the
collection of to figure out which partial to render. Active Record models 
inherit a method which returns a partial name based on the name of the model. 
However, it is possible to overwrite said behavior. Additionally plain ruby 
objects are able to implement the same behaviour.


{% highlight ruby %}
class Article < ActiveRecord::Base
  def to_partial_path
    "articles/summary"
  end
end

class Spaceship
  def name
    "Serenity"
  end

  def to_partial_path
    "transports/class/firefly"
  end
end
{% endhighlight %}

I found this to be extremely useful in displaying an event feed. I had a
collection of `events`, each of which had a specific type (Comment, Upload,
Star, etc.) and each type had a unique partial it needed to render. Using
`to_partial_path` I was to able avoid writing conditional code.

{% highlight ruby %}
class Event < ActiveRecord::Base
  def to_partial_path
    "events/#{type}"
  end
end

....
<%= render @events %>
{% endhighlight %}


