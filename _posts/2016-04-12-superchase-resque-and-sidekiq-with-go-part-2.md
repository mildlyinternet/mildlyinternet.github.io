---
layout: post
title: Supercharge Resque and Sidekiq With Go Part 2
category: code
description: "Process background job queues faster by offloading jobs to Go.
Take advantage of Go's speed and low memory usage to replace slow or IO bound
Ruby Workers. In Part 2 see some real world examples"
author: patrick
---

In [Part 1]({% post_url 2016-04-04-supercharge-resque-and-sidekiq-with-go %}) we
looked at a way to push jobs onto a queue with Ruby and have a Go program
consume them. In [Part 2]({% post_url 2016-04-12-superchase-resque-and-sidekiq-with-go-part-2 %})
I want to explore
some real world examples.

## Tracking & Analytics
Your PM has asked you plug in _another_ analytics service (on top of the 8 you
already have) to measure the usage of your application. Whenever a request
happens the relevant user details are logged. Email, `user_id`, url, ip address,
etc.

There is an existing Sidekiq worker that looks similar to this

~~~ruby
class TrackUserJob < ActiveJob::Base
  queue_as :track

  def perform(id, url, ip_address, controller, action)
    user = User.find(id)

    GenericService.track!(id: user.id, name: user.name, email: user.email, ip_address: ip_address, endpoint: "#{controller}/#{action}"
    SomethingFromGoogle.track(id: user.id, name: user.name, email: user.email, url: url)
    HotNewStartup.trendy_term_for_tracking("userId" => user.id, "userName" => user.name, "userEmail" => user.email)
    #...
    #...
    #...
    #...

  end
end
~~~

This sort of job is an excellent candidate to switch over to Go. There are no
application side-effects. Fire off a request and forget about it. It is also
probably high volume - a million plus a month.

There is one blocker to remove before we can switch - the user database lookup
`User.find(id)`. We could look up the user in our Go worker but I would
discourage that approach.  Doing so duplicates knowledge of the database schema.
If a column changes, two programs now need to be updated.

Furthermore, a method like `#name` by may not be a database column. It could be
a method on the ActiveRecord model which concatenates `first_name` and
`last_name`. Again we would be forced to duplicate that logic.

It is much easier to change the signature of the method. Rewrite it to just take
all the string values it needs.

~~~ruby
 # From
 def perform(id, url, ip_address, controller, action)

 # To
 def perform(id, name, email, url, ip_address, controller, action)
~~~

Now we have no need to query the database.


### Pushing the Job onto the Queue

There are a few options on how to push the job onto the queue.

1. Change every place the method is called to use `Sidekiq::Client.push` instead
of `TrackUserJob.perform_async(..args)`

2. Keep the `TrackUserJob` class but make it almost empty. Remove everything
except for `queue_as`

Although Option 1 might involve a bit of [shotgun
surgery](https://en.wikipedia.org/wiki/Shotgun_surgery) I tend to prefer it over
having an empty and somewhat confusing class.

~~~ruby
 # somewhere in a controller
 MyRubyJob.perform_async(current_user.id, current_user.name, current_user.email, current_path, request.ip_address, controller_name, action_name)

 # becomes
  Sidekiq::Client.push(
    "queue" => "track",
    "class" => "TrackUserJob",
    "args"  => [current_user.id, current_user.name, current_user.email, current_path, request.ip_address, controller_name, action_name]
  )
~~~


### Go Worker

Here is the code from [Part 1]({% post_url
2016-04-04-supercharge-resque-and-sidekiq-with-go %}). The only change so far
has been renaming the Worker function to `TrackUserJob`.

~~~ go
package main

import (
    "fmt"
    "github.com/jrallison/go-workers"
)

func TrackUserJob(message *workers.Msg) {
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

    // pull messages from "track" queue with concurrency of 1000
    workers.Process("track", TrackUserJob, 1000)

    // Blocks until process is told to exit via unix signal
    workers.Run()
}
~~~

To access the arguments for the job the `Msg` type provides an
[`Args`](https://godoc.org/github.com/jrallison/go-workers#Msg.Args) method which
returns a wrapper object around
[simplejson](https://github.com/bitly/go-simplejson).  To extract the arguments
array use the
[`Array`](https://godoc.org/github.com/bitly/go-simplejson#Json.Array) method from `simplejson`.


~~~go
func TrackUserJob(message *workers.Msg) {
 args, _ := message.Args().Array()
 // args looks like [1, "patrick", "patrick@devlocker.io", .....]

 // Omitted
 // Do stuff with args
 // Post to services
}
~~~

The arguments will come in the same order they were put in. If working with an
array feels clumsy here is an example of using hashes. In the Ruby code switch
the `args` to a hash.

~~~ruby
  Sidekiq::Client.push(
    "queue" => "track",
    "class" => "TrackUserJob",
    "args"  => [{
      id: current_user.id,
      name: current_user.name,
      email: current_user.email,
      ...
    }]
  )
~~~

Redis will serialize that hash into a JSON string. There are two ways to
work with the data. The first is to continue to use simplejson. The data is
going to be a one element array with a map as the only value.

~~~
  [{"id":5,"name":"Patrick","email":"patrick@devlocker.io"}]
~~~

Use the
[`GetIndex`](https://godoc.org/github.com/bitly/go-simplejson#Json.GetIndex)
method to pull out the first and only element. Then use other simplejson methods
to extract out values.


~~~go
func TrackUserJob(message *workers.Msg) {
  args, _ := message.Args().GetIndex(0)

  id, _ := args.Get("id").Int()
  name, _ := args.Get("name").String()
  email, _ := args.Get("email").String()
}
~~~

It is also possible to unmarshal the JSON into a struct.

~~~go
type Args struct {
    Id    string `json:"id"`
    Name  string `json:"name"`
    Email string `json:"email"`
}

func TrackUserJob(message *workers.Msg) {
    var payload []Args
    json.Unmarshal(message.Args().ToJson(), &payload)

    args := payload[0]

    // args.Id => 5
    // args.Name => "Patrick"
    // args.Email => "patrick@devlocker.io"
}
~~~

With the arguments in hand all that is left is to write the HTTP requests.
Becasuse they are not very interesting I have omitted the requests from the
examples.

That is all there is to this simple example. The Go Worker pulls off jobs, runs
them, and exits. The next example will show how to post some data back to the
Ruby process.


## Computation Heavy
This example will demonstrate how to offload some computationally heavy work to
Go. Once finished it will return the results back to our Ruby program.

Because I like using [Sudoku]({% post_url 2016-02-03-rust-with-ruby %}) as an
example, the Ruby program will be a Sudoku solving service. Users will post a
puzzle as a string and the program will solve it in the background and notify
the user when finished.

However, imagine the algorithm we use to solve the puzzles is not very fast
(short on time so it is an inelegant brute force solver).

Instead of having Ruby inefficiently try to finish the puzzles it will offload
the work to Go.  Each unsolved puzzle will get pushed onto a `unsolved_puzzles`
queue. Go Workers will pick up the jobs, compute the unsolved puzzle and push
the result onto a `solved` queue. Ruby workers will then pull off the solved
jobs and update the database record with the solved solution.


Here is the Ruby code


~~~ruby

# A Rails controller somewhere

class SudokuController < ApplicationController
  def create
    @sudoku = Sudoku.new(sudoku_params)

    if @sudoku.save
      Sidekiq::Client.push(
        "queue" => "unsolved_puzzles",
        "class" => "SudokuSolver",
        "args"  => [@sudoku.id.to_s, @sudoku.unsolved_puzzle]
      )
    else
      render :new
    end
  end

  private

  def sudoku_params
    params.require(:sudoku).permit(:unsolved_puzzle)
  end
end
~~~

Corresponding Go Code.

~~~ go
package main

import (
    "fmt"
    "github.com/jrallison/go-workers"
)

func Solve(puzzle string) string{
    // Code to solve a Sudoku puzzle
    // Returns the solved puzzle as a string
}

func SudokuSolver(message *workers.Msg) {
    args, _ := message.Args().Array()

    id := args[0]
    puzzle := args[1]

    solvedPuzzle := Solve(puzzle)

    // Add a job to the solved queue. Args need to be an array
    workers.Enqueue("solved", "SolvedPuzzleJob", []string{id, solvedPuzzle})
}

func main() {
    workers.Configure(map[string]string{
      // Omitted see above
    })

    // pull messages from "unsolved_puzzle" queue with concurrency of 100
    workers.Process("unsolved_puzzles", SudokuSolver, 100)

    // Blocks until process is told to exit via unix signal
    workers.Run()
}
~~~

The Go Worker pushes a job onto the solved queue with `workers.Enqueue`. The
first argument is the queue, the second is the class name and the third is the
array of arguments.

And a Sidekiq Job to save the result. Its class name must match the second
argument of `workers.Enqueue` from above.

~~~ruby
class SolvedPuzzleJob < ActiveJob::Base
  queue_as :solved

  def perform(id, solved_puzzle)
    sudoku = Sudoku.find(id)

    sudoku.update(solved_puzzle: solved_puzzle)
    sudoku.notify_user!
  end
end
~~~

## Things to consider

1. Redis is not meant to serialize large objects.

    If you try to avoid database lookups by dumping an entire object (or set of
    objects) into the arguments lists you may run into some performance issues. At
    that point either keep the job in Ruby or have it hit an API for the data
    instead.

2. Do not mix and match workers on the same queue

    Do not have both Sidekiq / Resque and Go working on the same queue. Not sure
    what will happen, but I cannot image it ending well.
