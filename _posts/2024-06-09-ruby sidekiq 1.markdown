---
layout: post
title:  "A study around Sidekiq's source code: poller, manager and processor"
date:   2024-06-09 12:00:20 +0200
categories: jekyll update
tags: sidekiq redis
---

# Sidekiq 

In Sidekiq, there are 3 important classes that we will study the source code

| Class                          | Role                                                         |
| :----------------------------- | ------------------------------------------------------------ |
| **Sidekiq::Scheduled::Poller** | check and fetch jobs from `scheduled` and `retry` jobs queue periodically.   |
| **Sidekiq::Manager**           | Create number of workers according to the parameter `concurrency` |
| **Sickie::Processor**          | exucute the job                                              |

**Synthesis**

`Sidekiq::Luancher` creates an instance of  `Sidekiq::Scheduled::Poller` and one of  `Sidekiq::Manager`, and calls their's  `start` method.

`Sidekiq::Scheduled::Poller`'s instance, after running it's start method, creates a loop thread that executes `enqueue -> wait`

`Sidekiq::Manager` ' instance, after running it's start method,  creates number of  `concurrency` workers (they are `Sidekiq::Processor`), and calls worker's start method.

Each `Sidekiq::Processor` 's instance , after running it's start method, create a new thead that executes `#process_one` repeatedly.



---

# Poller Manager and Processor 

Now let's go to check the source code.

When we start `sidekiq`, let's see the file `bin/sidekiq.rb`.

```ruby
begin
  cli = Sidekiq::CLI.instance
  cli.parse
  # parse parameters
  # ...
  cli.run
  # ...
end
```

let's go to the file `lib/sidekiq/cli.rb`

```ruby
def run(boot_app: true, warmup: true)
  # ...
  launch(self_read)
end

def launch(self_read)
  # ...
  @launcher = Sidekiq::Launcher.new(@config)
  begin
    launcher.run
   
  # ...
end
```

let's go to the file `lib/sidekiq/launcher.rb`

```ruby
def initialize(config, embedded: false)
  # ...
  @managers = config.capsules.values.map do |cap|
    Sidekiq::Manager.new(cap)
  end
  @poller = Sidekiq::Scheduled::Poller.new(@config)
end

def run(async_beat: true)
  # ...
  @thread = safe_thread("heartbeat", &method(:start_heartbeat)) if async_beat
  @poller.start
  @managers.each(&:start)
end
```

**There are 2 important lines**

```ruby 
@poller.start
@managers.each(&:start)
```

**We can see that `luancher` calls `poller` 's and `managers`' `start` method**





Now let's see the code of  `@poller.start` 

in the file `lib/sidekiq/scheduled.rb`, we can find `Sidekiq::Scheduled::Poller#start`

```ruby
class Poller
  # ...
  def start
    @thread ||= safe_thread("scheduler") {
      initial_wait

      until @done
        enqueue
        wait
      end
      logger.info("Scheduler exiting...")
    }
  end
  # ...
end
```

So this poller.start creates an infinite loop inside a new thread, then executes `enqueue -> wait` repeatedly.



in the file `lib/sidekiq/manager.rb`

```ruby
class Manager
  def initialize(capsule)
    @workers = Set.new
    @count.times do
      @workers << Processor.new(@config, &method(:processor_result))
    end
  end

  def start
    @workers.each(&:start)
  end
```



in the file `lib/sidekiq/processor.rb`

```ruby
class Processor
  def start
    @thread ||= safe_thread("#{config.name}/processor",&method(:run))
    # this thread calls private run method
  end
  
  private unless $TESTING
  def run
    # ...
    process_one until @done
    # ...
  end
```

This thread calls `run` method , that then calls `process_one` method.

---
### Dive into Sidekiq::Scheduled::Poller#start

let's refresh the code

```ruby
class Poller
  # ...
  def start
    @thread ||= safe_thread("scheduler") {
      initial_wait

      until @done
        enqueue
        wait
      end
      logger.info("Scheduler exiting...")
    }
  end
  # ...
end
```

```ruby
def initial_wait
  # Have all processes sleep between 5-15 seconds. 10 seconds to give time for
  # the heartbeat to register (if the poll interval is going to be calculated by the number
  # of workers), and 5 random seconds to ensure they don't all hit Redis at the same time.
  # ...
end
```

As the comment says

`initial_wait` is used to avoid Redis avalanche.

Now let's come back to see `enqueue`

```ruby
def enqueue
  @enq.enqueue_jobs
  # ...
end

def initialize
  @enq = (config[:scheduled_enq] || Sidekiq::Scheduled::Enq).new(config)
  # ...
end

SETS = %w[retry schedule]
def enqueue_jobs(sorted_sets = SETS)
  # A job's "score" in Redis is the time at which it should be processed.
  # Just check Redis for the set of jobs with a timestamp before now.
  redis do |conn|
    sorted_sets.each do |sorted_set|
      # Get next item in the queue with score (time to execute) <= now.
      # We need to go through the list one at a time to reduce the risk of something
      # going wrong between the time jobs are popped from the scheduled queue and when
      # they are pushed onto a work queue and losing the jobs.
      while !@done && (job = zpopbyscore(conn, keys: [sorted_set], argv: [Time.now.to_f.to_s]))
        @client.push(Sidekiq.load_json(job))
        logger.debug { "enqueued #{sorted_set}: #{job}" }
      end
    end
  end
end
```

We can understand that poller.start `fetch jobs` from Redis ZSET : `retry` and `schedule` `queue`,  and `push` them into client, that means a Redis FIFO dataset: normal `Sidekiq async queue` .



PS: when we use 

```ruby
Worker.perform_in(2.minutes)
```

**It does not mean that Sidekiq will execute our job in 2 minutes; it just means that the job will be fetched from schedule queue in 2 minutes and then pushed into the Sidekiq normal queue.**

**If our Sikekiq queue already is saturated, as this queue is FIFO by nature, we will need to wait.**



---

### Dive into Sidekiq::Processor#process_one

```ruby
# 'lib/sidekiq/processor.rb'

def process_one(&block)
  @job = fetch
  process(@job) if @job
  @job = nil
end

def fetch
  j = get_one
  if j && @done
    j.requeue
    nil
  else
    j
  end
end

def get_one
  uow = capsule.fetcher.retrieve_work
  uow
	# ...
end
```

```ruby
# 'lib/sidekiq/fetch.rb'

def retrieve_work
  qs = queues_cmd
  # ...
  queue, job = redis { |conn| conn.blocking_call(conn.read_timeout + TIMEOUT, "brpop", *qs, TIMEOUT) }
  UnitOfWork.new(queue, job, config) if queue
end
```

`brpop` is Redis's command that fetch the first element from a given list



Now let's see how Sidekiq execute job from different queues

```ruby
# 'lib/sidekiq/fetch.rb'


# Creating the Redis#brpop command takes into account any
# configured queue weights. By default Redis#brpop returns
# data from the first queue that has pending elements. We
# recreate the queue command each time we invoke Redis#brpop
# to honor weights and avoid queue starvation.
def queues_cmd
  if @strictly_ordered_queues
    @queues
  else
    permute = @queues.shuffle
    permute.uniq!
    permute
  end
end

```

**It the queue is strictly ordered , that Sidekiq will execute all jobs from the first queue, then the second and and so on.**

**it queue is not strictly ordered, Sidekiq will randomly permutate all queues according to their weight**  

---

&nbsp;
&nbsp;

After having seen the `fetch` method, lets come back to `Sidekiq::Processor#process_one` , and focus on `process` method

```ruby
# 'lib/sidekiq/processor.rb'

def process_one(&block)
  @job = fetch
  process(@job) if @job
  @job = nil
end

def process(uow)
  jobstr = uow.job
  queue = uow.queue_name
	# ...
  job_hash = Sidekiq.load_json(jobstr)
  # ...
        dispatch(job_hash, queue, jobstr) do |inst|
          config.server_middleware.invoke(inst, job_hash, queue) do
            execute_job(inst, job_hash["args"])
          end
        end
  # ...
end

def dispatch(job_hash, queue, jobstr)
	#...
  klass = Object.const_get(job_hash["class"])
  inst = klass.new
  yield inst
  #...
end


def execute_job(inst, cloned_args)
  inst.perform(*cloned_args)
end
```

We have finally seen the `inst.perform` method. 