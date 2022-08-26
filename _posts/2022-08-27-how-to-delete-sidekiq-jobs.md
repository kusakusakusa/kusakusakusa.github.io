---
title:  "How To Delete Sidekiq Jobs"
date:   2022-08-27 09:00:00
permalink: how-to-delete-sidekiq-jobs
---

# How To Delete Sidekiq Jobs

How to clear every single sidekiq related jobs and queues.

```ruby
require 'sidekiq/api'
Sidekiq::Queue.all.each(&:clear)
Sidekiq::RetrySet.new.clear
Sidekiq::ScheduledSet.new.clear
Sidekiq::DeadSet.new.clear
```
