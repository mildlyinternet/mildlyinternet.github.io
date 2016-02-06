---
layout: post
title: Conditionals Are the Screams of Missing Objects
category: code
---

In object oriented programming we dislike the idea of type checking -
`thing.is_a?(ClassWeKnowAbout)`. We prefer to just send messages to objects and
not care what class they are. In Ruby a conditional - `if / else / end` - is a
type check. Don't believe me? Consider the following code:

~~~ ruby
if @user.present?
  @user.name
else
  "Guest User"
end
~~~

At first glance all the code is doing is checking to see if something is
present. If it is, we execute one code path, otherwise we do something else. But
this is equivalent to:

~~~ ruby
if true # well, truthy
  @user.name
else
 "Guest User"
end
~~~

`true` in Ruby is in fact [an instance of
TrueClass](http://ruby-doc.org/core-2.2.0/TrueClass.html). So really what the
code is doing is this:

~~~ ruby
if @user.present?.is_a?(TrueClass)
  @user.name
else
 "Guest User"
end
~~~

Which is a type check.

Most of us learned to use `if` from procedural languages and its usage looks
completely normal here - but it's hampering our ability to use objects to their
full potential. In this example we used the `if` statement because a user may or
may not be present. `nil` is a possible value that we have to handle. The
easiest thing to do here was to just add the `if` and go on about our day.

However, conditionals have a tendency to multiply. Imagine the function above
was part of a class that was responsible for showing user information. That
`nil` is going to propagate its way through our code.

~~~ ruby
class UserPrinter
  def initialize(user)
    @user = user
  end

  def name
    if @user
      @user.name
    else
      "Guest User"
    end
  end

  def address
    if @user
      @user.address
    else
      "No Address"
    end
  end

  def phone
    if @user
      @user.phone(format: :international)
    else
      "No Phone Number"
    end
  end

  def email
    if @user
      @user.send_email
    else
      # Do nothing
    end
  end
end
~~~

We are trying to send `nil` a message here - we want to call `#name` on it but
can't because obviously `nil` does not have a `#name` method. But that repeated
`if / else / end` structure is screaming at us that something isn't right. If we
are trying to talk to `nil` then `nil` is definitely something. We just don't
have an object or a name for it at the moment.

In OO we want to send messages. OO is about being condition averse. So let's
give this `nil` a name. In this example no user is meant to be a "Guest
User". That is a thing, an object, a concept. So it should be a class. It's
going to implement the same API a user, because again, we just want to send
messages. The implementations of the of the methods may return something or do
nothing.

~~~ ruby
class GuestUser
  def name
    "Guest User"
  end

  def address
    "No Address"
  end

  def phone(options = {})
    "No Phone Number"
  end

  def send_email
  end
end
~~~

Now `UserPrinter` can become very simple. In the initializer if no user is
passed in we default to a `GuestUser`. We haven't broken any existing APIs -
`UserPrinter` still works the same as it did before but we have removed four
conditionals and made the code much more digestible.

~~~ ruby
class UserPrinter
  def initialize(user)
    @user = user || GuestUser.new
  end

  def name
    @user.name
  end

  def address
    @user.address
  end

  def phone
    @user.phone(format: :international)
  end

  def email
    @user.send_email
  end
end
~~~

This is known as the [Null Object
Pattern](https://en.wikipedia.org/wiki/Null_Object_pattern). If you find
yourself constantly checking for the `nil` values, try and replace some of them
with a Null Object.
