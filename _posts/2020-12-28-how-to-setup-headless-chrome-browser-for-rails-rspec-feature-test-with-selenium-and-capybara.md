---
title:  "How To Setup Headless Chrome Browser For Rails RSpec Feature Test With Selenium And Capybara"
date:   2020-12-28 09:00:00
permalink: how-to-setup-headless-chrome-browser-for-rails-rspec-feature-test-with-selenium-and-capybara
---

This is a concept in Ruby, at least for Ruby 2 before the [2020 Christmas present by Matz and team](https://www.ruby-lang.org/en/news/2020/12/25/ruby-3-0-0-released/), where threads can only run concurrently, but not in parallel.

This distinction needs to be clear.

In parallelism, multiple threads run together and doing stuff like iops, switching and allocating memory and variables, as well as making HTTP request.

On the other hand in concurrency, it is only when 1 thread becomes idle, maybe from making a HTTP request that will take at least finite amount of time to return, does another thread execute its program. This is the principle that ruby threads adhere to, at least if they are running under [Ruby MRI](https://en.wikipedia.org/wiki/Ruby_MRI) which is the default.

The reason for this design is to prevent race conditions and uphold thread safety, at the expense of parallelism and thus performance, which is totally understandable. And the guardian of this job is none other than the Global Interpreter Lock (GIL).

Note to self: The GIL is analogous to the event loop mechanism in the Javascript engine.

To illustrate this concept better, below is a simple sinatra application that opens up 2 routes. Both executes 2 commands of sleep, but one of them is done using threads.

```
require 'sinatra'

get '/sleep' do
  start_time = Time.now

  2.times do
    sleep 2
  end

  elapsed_time = Time.now - start_time
  elapsed_time.to_s
end

get '/sleep_with_thread' do
  start_time = Time.now

  threads = []
  2.times do
    threads << Thread.new do
      sleep 2
    end
  end

  threads.each(&:join)

  elapsed_time = Time.now - start_time
  elapsed_time.to_s
end
```

As sleep is an idle operation, we would expect the GIL to release the lock on the first thread and execute the next thread. This will result in the elapsed_time for the sleep_with_thread route to be around 2 seconds, while that for the sleep route will be accumulatively at around 4 seconds, as shown in the screen shots below.

INSERT IMAGE of output of both routes

However, things will behave differently if it was not an idle command like sleep. Let’s setup 2 routes that goes to work rather than sleep!

```ruby
require 'sinatra'
 get '/work' do
   start_time = Time.now
 2.times do
     string = ''
     50_000.times do
       string += '仕事!'.freeze
     end
   end
 elapsed_time = Time.now - start_time
   elapsed_time.to_s
 end

 get '/work_with_thread' do
   start_time = Time.now
 threads = []
   2.times do
     threads << Thread.new do
       string = ''
       50_000.times do
         string += '仕事!'.freeze
       end
     end
   end
 threads.each(&:join)
 elapsed_time = Time.now - start_time
   elapsed_time.to_s
 end
```
Note: The freeze command is a mere memory optimization and does not affect the response time in any significant manner.

The elapsed_time between the 2 operations are similar, as shown in the screenshots below.

INSERT IMAGE of output of both routes

This clearly depicts that the 2 threads does not run simultaneously, which would otherwise have resulted in the operation concluding in half the time shown.

Instead, the 2 threads are executed synchronously, one after another, because the operations involved are not idle.

This little experiment clearly defines the difference between concurrency versus parallelism, and portrays the limited capability of using threads in Ruby.

## Conclusion

Hence, only spawn new threads when there is an idle operation involved, with the most practical example being making a HTTP request, in order to take advantage of threading in ruby.

Otherwise, you will be doing a move as redundant as “taking off your pants to fart”.

