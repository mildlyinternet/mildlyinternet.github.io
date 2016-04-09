---
layout: post
title: Cleaning up your spec/rails helpers
category: code
description: "Organizing your RSpec configurations"
---

A common pattern when working on a Rails project is to add all RSpec
configuration to the `spec_helper` or `rails_helper`. Over time this will cause
these files to grow. Configurations lose context as to what they do.

An approach to cleaning these files up is to add configurations for specific
libraries in their own files.

Adding the following line in your `spec_helper` or `rails_helper` will load all
files in `spec/support`.

~~~ ruby
Dir[Rails.root.join("spec/support/**/*.rb")].each { |f| require f }
~~~

The next step is to move/create new configs in separate files in `spec/support`.
For example your database cleaner config can be stored in
`spec/support/database_cleaner.rb`.

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

An added benefit of this approach is that removing a test library is a simple
as `rm spec/support/some_library.rb` and removing the gem from the Gemfile.
