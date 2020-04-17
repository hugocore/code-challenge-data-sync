# code-challenge-data-sync

This was an example of a real-life-in-production service that I developed to sync data with 3rd parties in a company I worked for in the past. With this example, I want to demonstrate my mental process and capabilities when dealing with actual problems and show small snippets of my code. This code is meta and was redacted and refactored to prevent copyrights, so don't expect for it to run.

## Requirements

In this problem, we need a simple solution to keep a system's data in sync with
a 3rd party. Based on a real-life challenge, these were the requirements:

1. This 3rd party could be swapped out in the future
2. Respect the API throttling of one-hit per minute
3. When syncing the system cannot halt and lock

## Solution
 
Here's how I tackled every requirement:
 
### 1. This 3rd party could be swapped out in the future

I´ve decoupled the system logic from the 3rd party's specifics
by adding an abstraction layer made of adaptors that follow a
common interface. They respond to commands
known by the client and adapt to the 3rd party used at that time.

```ruby
class SyncDataService
  def self.call(...)
    new(...).call
  end

  def call(data)
    # Logic to decide which adatper to use, e.g.
    adapter = case data
      when ...
        XYZAdapter.new(data)
      else
        ThirdPartyAdapter.new(data)
    

    syncer = DataSyncer.new(adapter)

    syncer.sync
  end
end

class DataSyncer
  def initialize(adapter)
    @adapter = adapter
  end

  def sync
    @adapter.sync
  end
end

class ThirdPartyAdapter
  def initialize(data)
    @data = data
  end

  def sync
    parse(data)
  end

  private

  def parse(data)
    ...
  end
end

class XYZAdapter
  def initialize(data)
    @data = data
  end

  def sync
    # something else...
  end
end
```


### 2. Respect the API throttling of one-hit per minute

Besides running this as a Job every minute, I wanted to make sure
that if something fails we are not going retry endless, spamming
the queue with a cascade of retries or hitting the 3rd party threshold.

Hence, I made sure jobs didn't retry and that didn't get enqueued
again if one is already running, mistakenly or else. To track that,
every running job was temporarily stored in a redis cache. For every
instance of that job, the cache was checked to block a job from running
if there was already one (with the same params) running.

```ruby
class SyncJob < SidekiqJob
  include SafeJobs

  sidekiq_options retry: false

  SOME_ENDPOINT_PATH = "...".freeze


  def perform(timestamp)
    unique_job(key: "SyncJob_#{timestamp}") do
      SyncDataService.call(fetch_data)
    end
  end

  private

  def fetch_data
    client.get(SOME_ENDPOINT_PATH)
  end

  def client
    # Built the client to hit the current third party API
  end
end

class SafeJobs
  class AlreadyRunningError < StandardError; end

  TIMEOFF = # minute or more

  def unique_job(redis: REDIS, key:)
    redis_key = ["job_running", key].join(":")

    redis.with do |r|
      unless r.set(redis_key, 1, ex: TIMEOFF, nx: true) do
        raise AlreadyRunningError
      end

      begin
        yield
      ensure
        r.del(redis_key)
      end
    end
  end
end
```

### 3. When syncing the system cannot halt and lock

When syncing, I kept any side-effects async to prevent the
execution from stopping or taking too long. This way, was easier
to know exactly what these were and treat them differently
without impacting the execution time of every sync job.

For instance, say that for each deletion whilst syncing, 
another service needed to be notified while or after
the sync was completed. To cope with these effects
I've introduced async code in Ruby that would Publish-Subscribe
to events and react in separate scalable micro jobs, ran async, later on.

I've used https://github.com/krisleech/wisper
to provide the needed pub-sub capabilities in Ruby here.

```ruby
module DataParsers
  class DeleteService
    include Service

    def initialize(id)
      @id = id
    end

    def call(id)
      return unless record

      record.with_lock do
        return if record.in_state?("deleteled")

        record.transition_to! :deleteled

        broadcast(:mymodel_deleteled, id)
      end
    end

    private

    attr_reader :id

    def record
      @record ||= MyModel.find(id)
    end
  end
end

class MyModelListener
  def self.mymodel_deleteled(id)
    model = MyModel.find(id)

    Pusher.new.call(
      title: "deleteled",
      message: "#{model.something} was deleteled",
    )
  end
end

Rails.application.config.to_prepare do
  async_listeners = [
    MyModelListener
  ]

  async_listeners.each do |listener|
    Wisper.subscribe(listener, async: true)
  end
end
```
