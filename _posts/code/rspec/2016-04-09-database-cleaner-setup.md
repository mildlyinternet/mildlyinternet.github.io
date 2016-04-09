---
layout: post
title: Database Cleaner with RSpec
category: code
description: "Setting up Database Cleaner with RSpec"
---

When writing tests for a Rails application there is a good chance you will need
to create records in the database. To have reliable tests those records should
be removed after every run. This is a painful process to do manually. However
there is a library that handles cleaning up your database after every run.

Setting up [database_cleaner](https://github.com/DatabaseCleaner/database_cleaner)
can be a little tricky, especially if you are not using ActiveRecord.

## Installation and Basic Setup

Add the database_cleaner gem to your Gemfile:

~~~ruby
group :test do
  gem "database_cleaner"
end
~~~

The next step is configuring the DatabaseCleaner gem for your ORM. In the case of
ActiveRecord and the transaction strategy the configuration below is all you need.
Add the following to `spec/rails_helper.rb`

~~~ ruby
RSpec.configure do |config|
  config.use_transactional_fixtures = false

  config.before(:suite) do
    DatabaseCleaner.strategy = :transaction
    DatabaseCleaner.clean_with(:truncation)
  end

  config.around(:each) do |example|
    DatabaseCleaner.cleaning do
      example.run
    end
  end
end
~~~

DatabaseCleaner comes with several strategies for cleaning your database after
creating records. For SQL databases the fastest strategy will be transaction
as transactions are simply rolled back. Read more
[here](https://github.com/DatabaseCleaner/database_cleaner#what-strategy-is-fastest).

## Usage with Capybara

Using DatabaseCleaner with Capybara has a small gotcha. When using the transaction
strategy there can't be multiple database connections.

Most Capybara drivers run the application in a separate process that is isolated
from the running tests. Running the application in a separate process more
closely mirrors users using your application so there is value in having this
isolation.

The problem caused by multiple database connections is data created in one
connection's transaction is not accessible from another. The work around is to
use the truncation strategy only for feature tests. The truncation strategy is
much slower than the transaction strategy therefore the usage of truncation should
be limited.

The follow configuration sets the DatabaseCleaner strategy to truncation for
feature specs only.

~~~ ruby
require "capybara/rspec"

#...

RSpec.configure do |config|
  config.use_transactional_fixtures = false

  config.before(:suite) do
    DatabaseCleaner.clean_with(:truncation)
  end

  config.before(:each) do
    DatabaseCleaner.strategy = :transaction
  end

  config.before(:each, type: :feature) do
    DatabaseCleaner.strategy = :truncation
  end

  config.around(:each) do |example|
    DatabaseCleaner.cleaning do
      example.run
    end
  end
end
~~~

## Mongoid

When using Mongoid with Rails you will need to alter the strategy DatabaseCleaner
uses for removing data. Transactions are only available in SQL backed ORMs. The
following configuration should work for a Rails application that uses Mongoid.

~~~ ruby
RSpec.configure do |config|
  config.use_transactional_fixtures = false

  config.before(:suite) do
    DatabaseCleaner.strategy = :truncation
  end

  config.around(:each) do |example|
    DatabaseCleaner.cleaning do
      example.run
    end
  end
end
~~~
