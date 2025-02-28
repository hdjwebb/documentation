---
title: Ruby on Rails log collection
kind: documentation
aliases:
  - /logs/languages/ruby
further_reading:
- link: "https://github.com/roidrage/lograge"
  tag: "Github"
  text: "Lograge Documentation"
- link: "/logs/log_configuration/processors"
  tag: "Documentation"
  text: "Learn how to process your logs"
- link: "/logs/faq/log-collection-troubleshooting-guide/"
  tag: "FAQ"
  text: "Log Collection Troubleshooting Guide"
- link: "https://www.datadoghq.com/blog/managing-rails-application-logs/"
  tag: "Blog"
  text: "How to collect, customize, and manage Rails application logs"
- link: "https://www.datadoghq.com/blog/log-file-control-with-logrotate/"
  tag: "Blog"
  text: "How to manage log files using logrotate"
  
---

## Overview

To send your logs to Datadog, log to a file with [`lograge`][1] and tail this file with your Datadog Agent. When setting up logging with Ruby, keep in mind the [reserved attributes][2].

Instead of having a Rails logging output like this:

```text
Started GET "/" for 127.0.0.1 at 2012-03-10 14:28:14 +0100
Processing by HomeController#index as HTML
  Rendered text template within layouts/application (0.0ms)
  Rendered layouts/_assets.html.erb (2.0ms)
  Rendered layouts/_top.html.erb (2.6ms)
  Rendered layouts/_about.html.erb (0.3ms)
  Rendered layouts/_google_analytics.html.erb (0.4ms)
Completed 200 OK in 79ms (Views: 78.8ms | ActiveRecord: 0.0ms)
```

You can expect to see a log line with the following information in JSON format:

```json
{
  "timestamp": "2016-01-12T19:15:19.118829+01:00",
  "level": "INFO",
  "logger": "Rails",
  "method": "GET",
  "path": "/jobs/833552.json",
  "format": "json",
  "controller": "jobs",
  "action": "show",
  "status": 200,
  "duration": 58.33,
  "view": 40.43,
  "db": 15.26
}
```

## Setup

This section describes the minimum setup required to forward your Rails application logs to Datadog. For a more in-depth example of this setup, see [How to collect, customize, and manage Rails application logs][3].

1. Add the Lograge gem in your project:

    ```ruby
    gem 'lograge'
    ```

2. Configure Lograge. In your configuration file, set the following:

    ```ruby
    # Lograge config
    config.lograge.enabled = true

    # This specifies to log in JSON format
    config.lograge.formatter = Lograge::Formatters::Json.new

    ## Disables log coloration
    config.colorize_logging = false

    # Log to a dedicated file
    config.lograge.logger = ActiveSupport::Logger.new(Rails.root.join('log', "#{Rails.env}.log"))

    # This is useful if you want to log query parameters
    config.lograge.custom_options = lambda do |event|
        { :ddsource => 'ruby',
          :params => event.payload[:params].reject { |k| %w(controller action).include? k }
        }
    end
    ```

    **Note**: You can also ask Lograge to add contextual information to your logs. See [the Lograge documentation][4] for more details.

3. Configure your Datadog Agent. Create a `ruby.d/conf.yaml` file in your `conf.d/` folder with the following content:

    ```yaml
      logs:
        - type: file
          path: "<RUBY_LOG_FILE_PATH>.log"
          service: ruby
          source: ruby
          sourcecategory: sourcecode
          ## Uncomment the following processing rule for multiline logs if they
          ## start by the date with the format yyyy-mm-dd
          #log_processing_rules:
          #  - type: multi_line
          #    name: new_log_start_with_date
          #    pattern: \d{4}\-(0?[1-9]|1[012])\-(0?[1-9]|[12][0-9]|3[01])
    ```

    Learn more about the [Agent log collection][5].

4. [Restart the Agent][6].

## Getting further

### Connect logs and traces

If APM is enabled for this application, you can improve the connection between application logs and traces by [following the APM Ruby logging instructions][7] to automatically add trace and span IDs in your logs.

### Good logging practices in your application

Now that your logging configuration is sending proper JSON, you should use it as much as possible.

When logging, add as much context (user, session, action, and metrics) as possible.

Instead of logging simple string messages, you can use log hashes as shown in the following example:

```ruby
my_hash = {'user' => '1234', 'button_name'=>'save','message' => 'User 1234 clicked on button saved'};
logger.info(my_hash);
```

The hash is converted into JSON. Then, you can have Analytics for `user` and `button_name`:

```json
{
  "timestamp": "2016-01-12T19:15:18.683575+01:00",
  "level": "INFO",
  "logger": "WelcomeController",
  "message": {
    "user": "1234",
    "button_name": "save",
    "message": "User 1234 clicked on button saved"
  }
}
```

### RocketPants suggested logging configuration

In the `config/initializers/lograge_rocketpants.rb` file (it varies depending on your project), configure Lograge to work with `rocket_pants` controllers:

```ruby
# Come from here:
#   https://github.com/Sutto/rocket_pants/issues/111
app = Rails.application
if app.config.lograge.enabled
  ActiveSupport::LogSubscriber.log_subscribers.each do |subscriber|
    case subscriber
      when ActionController::LogSubscriber
        Lograge.unsubscribe(:rocket_pants, subscriber)
    end
  end
  Lograge::RequestLogSubscriber.attach_to :rocket_pants
end
```

### Grape's suggested logging configuration

Add the `grape_logging` gem:

```ruby
gem 'grape_logging'
```

Pass the additional configuration to Grape:

```ruby
use GrapeLogging::Middleware::RequestLogger,
      instrumentation_key: 'grape',
      include: [ GrapeLogging::Loggers::Response.new,
                 GrapeLogging::Loggers::FilterParameters.new ]
```

Create the `config/initializers/instrumentation.rb` file and add the following configuration:

```ruby
# Subscribe to grape request and log with a logger dedicated to Grape
grape_logger = Logging.logger['Grape']
ActiveSupport::Notifications.subscribe('grape') do |name, starts, ends, notification_id, payload|
    grape_logger.info payload
end
```

## Further Reading

{{< partial name="whats-next/whats-next.html" >}}

[1]: https://github.com/roidrage/lograge
[2]: /logs/log_configuration/attributes_naming_convention/#reserved-attributes
[3]: https://www.datadoghq.com/blog/managing-rails-application-logs
[4]: https://github.com/roidrage/lograge#installation
[5]: /agent/logs/
[6]: /agent/guide/agent-commands/#restart-the-agent
[7]: /tracing/other_telemetry/connect_logs_and_traces/ruby/
