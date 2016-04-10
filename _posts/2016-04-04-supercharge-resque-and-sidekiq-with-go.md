---
layout: post
title: Supercharge Resque and Sidekiq With Go
category: code
description: "Process background job queues faster by offloading jobs to Go.
Take advantage of Go's speed and low memory usage to replace slow or IO bound
Ruby Workers."
author: patrick
---

Process background job queues faster by offloading jobs to Go. Take advantage of
Go's speed and low memory usage to replace slow or IO bound Ruby Workers.

Certain types of jobs can be very computation heavy - calculating the product of
large matrices, solving puzzles ([sudoku]({% post_url 2016-02-03-rust-with-ruby
%})), etc. Or be very simple but occur very often. For example on every
page load you might create a job to post the user data and url to a third party
analytics service (this might happen a billion+ times a month).

If you are using Resque it will fork off processes similar in memory size to
your running application for each worker. Some large or old Rails apps have a
tendency to bloat well past 500MB which makes having many workers expensive. By
moving these types of jobs to Go, I free up my Ruby workers to do more
meaningful work related to my application.

## Setup Sidekiq

To keep things simple for this example I will run the Sidekiq UI from Rack
instead of Rails. To get the Sidekiq UI booted create a `config.ru` file with
following contents:

~~~ ruby
require "sidekiq"
require "sidekiq/web"

Sidekiq.configure_client do |config|
  config.redis = { db: 1 }
end

Sidekiq.configure_server do |config|
  config.redis = { db: 1 }
end

run Sidekiq::Web
~~~

To start the UI on port 4001 (can be any port) run:

~~~
rackup config.ru -p 4001
~~~

![]({{ site.url }}/images/supercharge-resque-and-sidekiq-with-go/sidekiq-ui-1.png)

Jobs need to be pushed onto a queue that the Go Workers will consume. The normal
Sidekiq API for pushing jobs is `MyRubyWorker.perform_async(args)`. The worker
lives in the Go code so that class does not exist within the scope of our Ruby
program. However, Sidekiq does provide an API for manually pushing jobs onto a
queue. Run this in an `irb` session:

~~~ ruby
Sidekiq::Client.push(
  "queue" => "go_queue",
  "class" => "GoWorker",
  "args"  => ["I am Running"]
)
~~~

Wrap that in a `1000.times do` block to quickly enqueue a large batch of jobs.


![]({{ site.url }}/images/supercharge-resque-and-sidekiq-with-go/sidekiq-ui-2.png)

## Go Worker

To create our Go Workers we'll use the excellent [Go
Workers](https://github.com/jrallison/go-workers) library. In the `main`
function the workers connect to Redis and are configured to pull from a
particular queue. The function name for the worker must match the `"class"`
argument we supplied earlier.

~~~ go
package main

import (
    "fmt"
    "github.com/jrallison/go-workers"
)

func GoWorker(message *workers.Msg) {
    fmt.Println(message)
}

func main() {
    workers.Configure(map[string]string{
        // location of redis instance
        "server": "localhost:6379",
        // instance of the database
        "database": "1",
        // number of connections to keep open with redis
        "pool": "10",
        // unique process id for this instance of workers (for proper recovery of inprogress jobs on crash)
        "process": "1",
    })

    // pull messages from "go_queue" with concurrency of 10
    workers.Process("go_queue", GoWorker, 10)

    // Blocks until process is told to exit via unix signal
    workers.Run()
}
~~~

Once that program starts running it will process any jobs we push onto the
`go_queue`. They will be processed extremely quickly while using minimal system
resources. Concurrency can easily be bumped up to several thousand which will
greatly out perform Sidekiq or Resque.

In Part 2 we'll take a look at some more advanced examples.
