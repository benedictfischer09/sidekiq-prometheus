# SidekiqPrometheus

Prometheus Instrumentation for Sidekiq.

* Sidekiq server middleware for reporting job metrics
* Global metrics reporter using the Sidekiq API for reporting Sidekiq cluster stats (requires Sidekiq::Enterprise)
* Sidecar Rack server to provide scrape-able endpoint for Prometheus


## Installation

Add this line to your application's Gemfile:

```ruby
gem 'sidekiq_prometheus'
```

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install sidekiq_prometheus

## Usage

To run with the defaults add this to your Sidekiq initializer

```
SidekiqPrometheus.setup
```

This will register metrics, start the global reporter (if available), and start the Rack server for scraping. The default port is 9357 but this is easily configurable.

If you are running multiple services that will be reporting Sidekiq metrics you will want to take advantage of the `base_labels` configuration option. For example:

```
SidekiqPrometheus.configure do |config|
  config.base_labels  = { service: 'image_api' }
  config.metrics_port = 9090
end

# always call setup after configure.
SidekiqPrometheus.setup
```

The call to `setup` is necessary as that it what registers the metrics and instruments Sidekiq.
There is also a helper method: `SidekiqPrometheus.configure!` which can be used to configure and setup in one step:

```
SidekiqPrometheus.configure! do |config|
  config.base_labels  = { service: 'dogs_api' }
end

# No need to call SidekiqPrometheus.setup because the bang method did it for us/
```

Once sidekiq server is running you can see your metrics or scrape them with Prometheus:

```
curl http://localhost:8675/metrics
```

#### Configuration options

* `base_labels`: Hash of labels that will be included with every metric when they are registered.
* `gc_metrics_enabled`: Boolean that determines whether to record object allocation metrics per job. The default is `true`. Setting this to `false` if you don't need this metric.
* `global_metrics_enabled`: Boolean that determines whether to report global metrics from the PeriodicMetrics reporter. When `true` this will report on a number of stats from the Sidekiq API for the cluster. This requires Sidekiq::Enterprise as the reporter uses the leader election functionality to ensure that only one worker per cluster is reporting metrics.
* `periodic_metrics_enabled`: Boolean that determines whether to run the periodic metrics reporter. `PeriodicMetrics` runs a separate thread that reports on global metrics (if enabled) as well worker GC stats (if enabled). It reports metrics on the interval defined by `periodic_reporting_interval`. Defatuls to `true`.
* `periodic_reporting_interval`: interval in seconds for reporting periodic metrics. Default: `30`
* `metrics_port`: Port on which the rack server will listen. Defaults to `9357`

```
SidekiqPrometheus.configure do |config|
  config.base_labels                   = { service: 'myapp' }
  config.gc_metrics_enabled            = false
  config.global_metrics_enabled        = true
  config.periodic_metrics_enabled      = true
  config.periodic_reporting_interval   = 20
  config.metrics_port                  = 8675
end
```

Custom labels may be added by defining the `prometheus_labels` method in the worker class:

```
class SomeWorker
  include Sidekiq::Worker

  def prometheus_labels
    { some: 'label' }
  end
end
```

## Metrics

### JobMetrics

All Sidekiq job metrics are reported with these labels:

* `class`: Sidekiq worker class name
* `queue`: Sidekiq queue name

| Metric | Type | Description |
|--------|------|-------------|
| sidekiq_job_count | counter | Count of Sidekiq jobs |
| sidekiq_job_duration | histogram | Sidekiq job processing duration |
| sidekiq_job_success | counter | Count of successful Sidekiq jobs |
| sidekiq_job_allocated_objects | histogram | Count of ruby objects allocated by a Sidekiq job |
| sidekiq_job_failed | counter | Count of failed Sidekiq jobs |

Notes:

* when a job fails only `sidekiq_job_count` and `sidekiq_job_failed` will be reported.
* `sidekiq_job_allocated_objects` will only be reported if `SidekiqPrometheus.gc_metrics_enabled? == true`

### Periodic GC Metrics

These require `SidekiqPrometheus.gc_metrics_enabled? == true` and `SidekiqPrometheus.periodic_metrics_enabled? == true`

| Metric | Type | Description |
|--------|------|-------------|
| sidekiq_allocated_objects | counter | Count of allocated objects by the worker |
| sidekiq_heap_free_slots | gauge | Number of free heap slots as reported by GC.stat |
| sidekiq_heap_live_slots | gauge | Number of live heap slots as reported by GC.stat |
| sidekiq_major_gc_count | counter | Count of major GC runs |
| sidekiq_minor_gc_count | counter | Count of minor GC runs |
| sidekiq_rss | gauge | RSS memory usage for worker process |

### Periodic Global Metrics

These require `SidekiqPrometheus.global_metrics_enabled? == true` and `SidekiqPrometheus.periodic_metrics_enabled? == true`

Periodic metric reporting relies onSidekiq Enterprise's leader election functionality ([Ent Leader Election ](https://github.com/mperham/sidekiq/wiki/Ent-Leader-Election))
which ensures that metrics are only reported once per cluster.

| Metric | Type | Description |
|--------|------|-------------|
| sidekiq_workers_size | gauge | Total number of workers processing jobs |
| sidekiq_dead_size | gauge | Total Dead Size |
| sidekiq_enqueued | gauge | Total Size of all known queues |
| sidekiq_queue_latency | summary | Latency (in seconds) of all queues |
| sidekiq_failed | gauge | Number of job executions which raised an error |
| sidekiq_processed | gauge | Number of job executions completed (success or failure) |
| sidekiq_retry_size | gauge | Total Retries Size |
| sidekiq_scheduled_size | gauge | Total Scheduled Size |
| sidekiq_redis_connected_clients | gauge | Number of clients connected to Redis instance for Sidekiq |
| sidekiq_redis_used_memory | gauge | Used memory from Redis.info
| sidekiq_redis_used_memory_peak | gauge | Used memory peak from Redis.info |
| sidekiq_redis_keys | gauge | Number of redis keys |
| sidekiq_redis_expires | gauge | Number of redis keys with expiry set |

The global metrics are reported with the only the `base_labels` with the exception of `sidekiq_enqueued` which will add a `queue` label and record a metric per Sidekiq queue.

## Development

After checking out the repo, run `bin/setup` to install dependencies. Then, run `rake test` to run the tests. You can also run `bin/console` for an interactive prompt that will allow you to experiment.

To install this gem onto your local machine, run `bundle exec rake install`.

## Contributing

Bug reports and pull requests are welcome on GitHub at https://github.com/fastly/sidekiq_prometheus. This project is intended to be a safe, welcoming space for collaboration, and contributors are expected to adhere to the [Contributor Covenant](http://contributor-covenant.org) code of conduct.

## Copyright

Copyright 2019 Fastly, Inc.

## License

The gem is available as open source under the terms of the [MIT License](https://opensource.org/licenses/MIT).

## Code of Conduct

Everyone interacting in the SidekiqPrometheus project’s codebases, issue trackers, chat rooms and mailing lists is expected to follow the [code of conduct](https://github.com/[USERNAME]/sidekiq_prometheus/blob/master/CODE_OF_CONDUCT.md).