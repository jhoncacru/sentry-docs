---
title: Configuration
sidebar_order: 1
---

Configuration is passed as part of the client initialization:

```javascript
Raven.configure do |config|
  config.dsn = '___DSN___'
  config.attr = 'value'
end
```

## Optional settings

`async`

: When an error or message occurs, the notification is immediately sent to Sentry. Raven can be configured to send asynchronously:

  ```ruby
  config.async = lambda { |event|
    Thread.new { Raven.send_event(event) }
  }
  ```

  Using a thread to send events will be adequate for truly parallel Ruby platforms such as JRuby, though the benefit on MRI/CRuby will be limited.

  The example above is extremely basic. For example, exceptions in Rake tasks will not be reported because the Rake task will probably exit before the thread can completely send the event to Sentry. Threads also won’t report any exceptions raised inside of them, so be careful!

  If the async callback raises an exception, Raven will attempt to send synchronously.

  We recommend creating a background job, using your background job processor, that will send Sentry notifications in the background. Rather than enqueuing an entire Raven::Event object, we recommend providing the Hash representation of an event as a job argument. Here’s an example for ActiveJob:

  ```ruby
  config.async = lambda { |event|
    SentryJob.perform_later(event.to_hash)
  }
  ```

  ```ruby
  class SentryJob < ActiveJob::Base
    queue_as :default

    def perform(event)
      Raven.send_event(event)
    end
  end
  ```

`encoding`

: While unlikely that you’ll need to change it, by default Raven compresses outgoing messages with gzip. This has a slight impact on performance, but due to the size of many Ruby stacktrace it’s required for the serve to accept the content.

  To disable gzip, set the encoding to ‘json’:

  ```ruby
  config.encoding = 'json'
  ```

`environments`

: As of v0.10.0, events will be sent to Sentry in all environments. If you do not wish to send events in an environment, we suggest you unset the SENTRY_DSN variable in that environment.

  Alternately, you can configure Raven to run only in certain environments by configuring the environments whitelist. For example, to only run Sentry in production:

  ```ruby
  config.environments = %w[ production ]
  ```

  Sentry automatically sets the current environment to RAILS_ENV, or if it is not present, RACK_ENV. If you are using Sentry outside of Rack or Rails, or wish to override environment detection, you’ll need to set the current environment by setting SENTRY_CURRENT_ENV or configuring the client yourself:

  ```ruby
  config.current_environment = 'my_cool_environment'
  ```

`excluded_exceptions`

: If you never wish to be notified of certain exceptions, specify ‘excluded_exceptions’ in your config file.

  In the example below, the exceptions Rails uses to generate 404 responses will be suppressed.

  ```ruby
  config.excluded_exceptions += ['ActionController::RoutingError', 'ActiveRecord::RecordNotFound']
  ```

  You can find the list of exceptions that are excluded by default in `Raven::Configuration::IGNORE_DEFAULT`. It is suggested that you append to these defaults rather than overwrite them with `=`.

`logger`

: The logger used by Sentry. Default is an instance of Raven::Logger.

  ```ruby
  config.logger = Raven::Logger.new(STDOUT)
  ```

  Raven respects logger levels.

`processors`

: If you need to sanitize or pre-process (before its sent to the server) data, you can do so using the Processors implementation. By default, a few processors are installed. The most important is `Raven::Processor::SanitizeData`, which will attempt to sanitize keys that match various patterns (e.g. password) and values that resemble credit card numbers.

  In your Sentry UI, data which has been sanitized will appear as “********” (or 0, if the value was an Integer).

  To specify your own (or to remove the defaults), simply pass them with your configuration:

  ```ruby
  config.processors = [MyOwnProcessor]
  ```

  Check out `Raven::Processor::SanitizeData` to see how a Processor is implemented.

  You can also specify values to be sanitized. Any strings matched will be replaced with the string mask (********). One good use for this is to copy Rails’ filter_parameters:

  ```ruby
  config.sanitize_fields = Rails.application.config.filter_parameters.map(&:to_s)
  ```

  The client scrubs the HTTP “Authorization” header of requests before sending them to Sentry, to prevent sensitive credentials from being sent. You can specify additional HTTP headers to ignore:

  You can also provide regex-like strings to the sanitizer:

  ```ruby
  config.sanitize_fields = ["my_field", "foo(.*)?bar]
  ```

  It’s also possible to remove HTTP header values which match a list:

  ```ruby
  config.sanitize_http_headers = ["Via", "Referer", "User-Agent", "Server", "From"]
  ```

  For more information about HTTP headers which may contain sensitive information in your application, see [RFC 2616](https://www.w3.org/Protocols/rfc2616/rfc2616-sec15.html).

  By default, Sentry sends up a stacktrace with an exception. This stacktrace may contain data which you may consider to be sensitive, including lines of source code, line numbers, module names, and source paths. To wipe the stacktrace from all error reports, require and add the RemoveStacktrace processor:

  ```ruby
  require 'raven/processor/removestacktrace'

  Raven.configure do |config|
    config.processors << Raven::Processor::RemoveStacktrace
  end
  ```

  By default, Sentry does not send POST data or cookies if present. To re-enable, remove the respective processor from the chain:

  ```ruby
  Raven.configure do |config|
    config.processors -= [Raven::Processor::PostData] # Do this to send POST data
    config.processors -= [Raven::Processor::Cookies] # Do this to send cookies by default
  end
  ```

`proxy`

: A string with the URL of the HTTP proxy to be used.

  ```ruby
  config.proxy = 'http://path.to.my.proxy.com'
  ```

`rails_report_rescued_exceptions`

: Rails catches exceptions in the ActionDispatch::ShowExceptions or ActionDispatch::DebugExceptions middlewares, depending on the environment. When _rails_report_rescued_exceptions_ is true (it is by default), Raven will report exceptions even when they are rescued by these middlewares.

  If you are using a custom exceptions app, you may wish to disable this behavior:

  ```ruby
  config.rails_report_rescued_exceptions = false
  ```

`release`

: Track the version of your application in Sentry.

  We guess the release intelligently in the following order of preference:

  -   Commit SHA of the last commit (git)
  -   Reading from the REVISION file in the app root
  -   Heroku’s dyno metadata (must have enabled via Heroku Labs)

  ```ruby
  config.release = '721e41770371db95eee98ca2707686226b993eda'
  ```

`sample_rate`

: The sampling factor to apply to events. A value of 0.00 will deny sending any events, and a value of 1.00 will send 100% of events.

  ```ruby
  # send 50% of events
  config.sample_rate = 0.5
  ```

`should_capture`

: By providing a proc or lambda, you can control what events are captured. A String (if you’ve captured a message) or the Exception (if you’ve captured an exception) will be passed to the Proc or lambda you provide - returning false will stop the event from sending to Sentry:

  ```ruby
  config.should_capture = Proc.new { |e| true unless e.contains_sensitive_info? }
  ```

`silence_ready`

: Upon start, Raven will write the following message to the log at the INFO level:

  `` ** [out :: hostname.example.com] I, [2014-07-22T15:32:57.498368 #30897] INFO – : ** [Raven] Raven 0.9.4 ready to catch errors” ``

  You can turn off this message:

  ```ruby
  config.silence_ready = true
  ```

`ssl_verification`

: By default SSL certificate verification is enabled in the client. It can be disabled.

  ```ruby
  config.ssl_verification = false
  ```

`tags`

: Default tags to send with each event.

  ```ruby
  config.tags = { foo: :bar }
  ```

`transport_failure_callback`

: If the transport fails to send an event to Sentry for any reason (either the Sentry server has returned a 4XX or 5XX response), this Proc or lambda will be called.

  ```ruby
  config.transport_failure_callback = lambda { |event|
    AdminMailer.email_admins("Oh god, it's on fire!").deliver_later
  }
  ```

## Environment Variables

`SENTRY_DSN`

: After you complete setting up a project, you’ll be given a value which we call a DSN, or Data Source Name. It looks a lot like a standard URL, but it’s actually just a representation of the configuration required by Raven (the Sentry client). It consists of a few pieces, including the protocol, public and secret keys, the server address, and the project identifier.

  With Raven, you may either set the SENTRY_DSN environment variable (recommended), or set your DSN manually in a config block:

  ```ruby
  # in Rails, this might be in config/initializers/sentry.rb
  Raven.configure do |config|
    config.dsn = 'http://public:secret@example.com/project-id'
  end
  ```
