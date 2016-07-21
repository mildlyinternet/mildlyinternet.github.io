---
layout: post
title: Clean & Simple FactoryGirl Definitions
category: code
description: "Using FactoryGirl to make test data creation simple."
---

## Clean & Simple FactoryGirl Definitions

FactoryGirl is a test data creation library for creating test data in a
structured and repeatable way. It allows you to succinctly express what types
of data your tests need and build them without repeating yourself.

### The Factories File

The core component of FactoryGirl is the factory definition. A factory
definition assigns default values for columns to a model when creating the
model in a test. The creators recommend sticking all your factory definitions
in one `factories.rb` file, but some apps will be so large that it becomes
unwieldy. FactoryGirl will autoload factories defined in `spec/factories/` and
`spec/factories.rb`; for our small blog app this one file will be fine.

~~~ ruby
# spec/factories.rb

FactoryGirl.define do
  factory :user do
    first_name "Example"
    last_name "User"
    email "user@example.com"
    joined_on Date.today
  end
end
~~~

~~~ ruby
# spec/models/user_spec.rb

require 'spec_helper'

RSpec.describe User do
  let(:user) { FactoryGirl.build(:user) }

  # ...
end
~~~

This is a simple factory definition, we just assign default values to each
attribute of `User`. If we want to overwrite any attributes in our test, pass a
hash of the attributes to `FactoryGirl.build` as the second argument.

~~~ ruby
FactoryGirl.build(:user, first_name: "Joe")
~~~

### Sequences

This is nice, but most apps get a lot more complicated than that. Let's say we
have a unique constraint on the user's email. If we try to create more than one
`User` with the default values, it'll error on the unique constraint! So rather
than specifying a different email in every single test, FactoryGirl provides
`sequences` for attributes.

~~~ ruby
# spec/factories.rb

FactoryGirl.define do
  factory :user do
    first_name "Example"
    last_name "User"
    sequence :email do |i|
      "user#{i}@example.com"
    end
    joined_on Date.today
  end
end
~~~

Now when a user is generated in your tests, the email attribute will increment,
ensuring that the model will always satisfy the unique constraint when using
default values.

~~~ ruby
FactoryGirl.build(:user) # => user1.example.com
FactoryGirl.build(:user) # => user2.example.com
~~~

### Traits

Another concept often repeated in tests is the different roles an object can
fill. For example, a User with no `joined_on` date might be considered a guest
user, or a user that hasn't signed up yet. Most of your tests will contain a
test context for "as a guest user", wherein you need to create a guest user.
FactoryGirl supplies `traits` just for this scenario.

~~~ ruby
# spec/factories.rb

FactoryGirl.define do
  factory :user do
    first_name "Example"
    last_name "User"
    sequence :email do |i|
      "user#{i}@example.com"
    end
    joined_on Date.today

    trait :guest do
      joined_on nil
    end
  end
end
~~~

~~~ ruby
# spec/controllers/some_controller_spec.rb

require 'spec_helper'

RSpec.describe SomeController do
  context "as a guest user" do
    let(:guest_user) { FactoryGirl.build(:user, :guest) }
  end
end
~~~

With one extra parameter you can specify roles for your objects. This adds
clarity for your tests and keeps them from getting cluttered with data. Traits
can also be mixed and matched for building complex scenarios, but for any
overlapping attributes, the last trait applied will win out.

~~~ ruby
FactoryGirl.build(:user, :guest, :last_logged_in_a_week_ago, :has_bad_email)
~~~

### Associations

Another major part of creating test data is associations, the way that data
relates to other data. Assuming that we have the following schema:

~~~ ruby
class User < ActiveRecord::Base
  has_many: :posts
  # The comments a user leaves on other people's posts
  has_many: :comments
end

class Post < ActiveRecord::Base
  belongs_to :author, class_name: "User"
  has_many :comments
end

class Comment < ActiveRecord::Base
  belongs_to :author, class_name: "User"
end
~~~

Here's how we would lay that out in FactoryGirl.

(The user factory stays the same)

~~~ ruby
factory :post do
  title "My First Post"
  content "Posting is really fun"
  posted_on Date.today
  association :author, factory: :user
end
~~~

In the post factory we define an author association. The name of the factory
needs to be supplied because the name of the association does not match a
defined factory, which we do by passing `factory: :user`.

~~~ ruby
factory :comment do
  content "I love this post!"
  association :author, factory: :user
  post
end
~~~

In the comment factory we supply the same author association, but we also
supply `post`. When defining the post association the name matches a factory,
so FactoryGirl can automatically find everything on its own.

Note: One way to clean up this example is to define what I call an alias
factory, which just points to the user factory.

~~~ ruby
factory :author do
  user
end
~~~

Then, in the post and comment factories, we can just reference `author`,
because now there's a matching factory.

### build vs create

Throughout this tutorial we've used `FactoryGirl.build` in all our examples.
This is analogous to `ActiveRecordObject.new`, it initializes the model without
persisting it to the database. Creating data is slow, wherever we can avoid
persistence we should in order to keep our test suite fast and responsive. When
you do need to persist your data, , you can use `FactoryGirl.create`. This is
analogous to `ActiveRecordObject.create`, which initializes the model and saves
it off.

You can read more about FactoryGirl at the following links:

https://github.com/thoughtbot/factory_girl
http://www.rubydoc.info/gems/factory_girl/file/GETTING_STARTED.md
