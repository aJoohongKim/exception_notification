Exception Notifier Plugin for Rails
====

[![Gem Version](https://fury-badge.herokuapp.com/rb/exception_notification.png)](http://badge.fury.io/rb/exception_notification)
[![Travis](https://api.travis-ci.org/smartinez87/exception_notification.png)](http://travis-ci.org/smartinez87/exception_notification)
[![Code Climate](https://codeclimate.com/github/smartinez87/exception_notification.png)](https://codeclimate.com/github/smartinez87/exception_notification)
![project status](http://stillmaintained.com/smartinez87/exception_notification.png)


The Exception Notifier plugin provides a mailer object and a default set of
templates for sending email notifications when errors occur in a Rails
application.

The email includes information about the current request, session, and
environment, and also gives a backtrace of the exception.

There's a great [Railscast about Exception Notifications](http://railscasts.com/episodes/104-exception-notifications-revised)
you can see that may help you getting started.

[Follow us on Twitter](https://twitter.com/exception_notif) to get updates and notices about new releases.

Installation
---

You can use the latest ExceptionNotification gem with Rails 3, by adding
the following line in your Gemfile

```ruby
gem 'exception_notification'
```

As of Rails 3 ExceptionNotification is used as a rack middleware, so you can
configure its options on your config.ru file, or in the environment you
want it to run. In most cases you would want ExceptionNotification to
run on production. You can make it work by

```ruby
Whatever::Application.config.middleware.use ExceptionNotifier,
  :email => {
    :email_prefix => "[Whatever] ",
    :sender_address => %{"notifier" <notifier@example.com>},
    :exception_recipients => %w{exceptions@example.com}
  }
```

ActionMailer Configuration
---

For the email to be sent, there must be a default ActionMailer delivery_method setting configured.
If you do not have one, you can use the following code (assuming your app server machine has sendmail).
Depending on the environment you want ExceptionNotification to run in, put the following code in your
config/production.rb and/or config/development.rb:

```ruby
config.action_mailer.delivery_method = :sendmail
# Defaults to:
# config.action_mailer.sendmail_settings = {
#   :location => '/usr/sbin/sendmail',
#   :arguments => '-i -t'
# }
config.action_mailer.perform_deliveries = true
config.action_mailer.raise_delivery_errors = true
```````


Campfire Notifier
---

Additionally, ExceptionNotification supports sending notifications to
your Campfire room.

First you'll need to add [tinder](https://github.com/collectiveidea/tinder)
to your `Gemfile`:

```ruby
gem 'tinder'
```````

To configure it, you need to set the subdomain, token and room name,
like this:

```ruby
Whatever::Application.config.middleware.use ExceptionNotifier,
  :email => {
    :email_prefix => "[Whatever] ",
    :sender_address => %{"notifier" <notifier@example.com>},
    :exception_recipients => %w{exceptions@example.com}
  },
  :campfire => {
    :subdomain => 'my_subdomain',
    :token => 'my_token',
    :room_name => 'my_room'
  }
```````

For more options to set Campfire, like _ssl_, check
[here](https://github.com/collectiveidea/tinder/blob/master/lib/tinder/campfire.rb#L17).


Webhook Notifier
---

ExceptionNotifier also can ship notifications over HTTP protocol.

For this, you'll need to add [HTTParty](https://github.com/jnunemaker/httparty)
to your `Gemfile`:

```ruby
gem 'httparty'
```````

To configure it, you need to set the url like this:

```ruby
Whatever::Application.config.middleware.use ExceptionNotifier,
  :email => {
    :email_prefix => "[Whatever] ",
    :sender_address => %{"notifier" <notifier@example.com>},
    :exception_recipients => %w{exceptions@example.com}
  },
  :webhook => {
    :url => 'http://domain.com:5555/hubot/path'
  }
```````

By default, the WebhookNotifier will call the URLs using the POST method.
But, you can change this using the `http_method` option.

```ruby
Whatever::Application.config.middleware.use ExceptionNotifier,
  :email => {
    :email_prefix => "[Whatever] ",
    :sender_address => %{"notifier" <notifier@example.com>},
    :exception_recipients => %w{exceptions@example.com}
  },
  :webhook => {
    :url => 'http://domain.com:5555/hubot/path',
    :http_method => :get
  }
```````

Besides the `url` and `http_method` options, all the other options are passed directly to HTTParty.
Thus, if the HTTP server requires authentication, you can include the following options:

```ruby
Whatever::Application.config.middleware.use ExceptionNotifier,
  :email => {
    :email_prefix => "[Whatever] ",
    :sender_address => %{"notifier" <notifier@example.com>},
    :exception_recipients => %w{exceptions@example.com}
  },
  :webhook => {
    :url => 'http://domain.com:5555/hubot/path',
    :basic_auth => {
      :username => 'alice',
      :password => 'password'
    }
  }
```````

For more HTTParty options, check out the [documentation](https://github.com/jnunemaker/httparty).


Customization
---

### Sections

By default, the notification email includes four parts: request, session,
environment, and backtrace (in that order). You can customize how each of those
sections are rendered by placing a partial named for that part in your
app/views/exception_notifier directory (e.g., _session.rhtml). Each partial has
access to the following variables:

```ruby
@kontroller     # the controller that caused the error
@request        # the current request object
@exception      # the exception that was raised
@backtrace      # a sanitized version of the exception's backtrace
@data           # a hash of optional data values that were passed to the notifier
@sections       # the array of sections to include in the email
```

Background views will not have access to @kontroller and @request.

You can reorder the sections, or exclude sections completely, by altering the
ExceptionNotifier.sections variable. You can even add new sections that
describe application-specific data--just add the section's name to the list
(wherever you'd like), and define the corresponding partial.

```ruby
#Example with two new added sections
Whatever::Application.config.middleware.use ExceptionNotifier,
  :email => {
    :email_prefix => "[Whatever] ",
    :sender_address => %{"notifier" <notifier@example.com>},
    :exception_recipients => %w{exceptions@example.com},
    :sections => %w{my_section1 my_section2} + ExceptionNotifier::Notifier.default_sections
  }
```

Place your custom sections under `./app/views/exception_notifier/` with the suffix `.text.erb`, e.g.
`./app/views/exception_notifier/_my_section1.text.erb`.

If your new section requires information that isn't available by default, make sure
it is made available to the email using the exception_data macro:

```ruby
class ApplicationController < ActionController::Base
  before_filter :log_additional_data
  ...
  protected
    def log_additional_data
      request.env["exception_notifier.exception_data"] = {
        :document => @document,
        :person => @person
      }
    end
  ...
end
```

In the above case, @document and @person would be made available to the email
renderer, allowing your new section(s) to access and display them. See the
existing sections defined by the plugin for examples of how to write your own.

You may want to include different sections for background notifications:

```ruby
#Example with two new added sections
Whatever::Application.config.middleware.use ExceptionNotifier,
  :email => {
    :email_prefix => "[Whatever] ",
    :sender_address => %{"notifier" <notifier@example.com>},
    :exception_recipients => %w{exceptions@example.com},
    :background_sections => %w{my_section1 my_section2} + ExceptionNotifier::Notifier.default_background_sections
  }
```

By default, the backtrace and data sections are included in background
notifications.

### Ignore Exceptions

You can choose to ignore certain exceptions, which will make
ExceptionNotifier avoid sending notifications for those specified.
There are three ways of specifying which exceptions to ignore:

- `:ignore_exceptions` - By exception class (i.e. ignore RecordNotFound ones)

- `:ignore_crawlers`   - From crawler (i.e. ignore ones originated by Googlebot)

- `:ignore_if`         - Custom (i.e. ignore exceptions that satisfy some condition)

---

* _:ignore_exceptions_

Ignore specified exception types.
To achieve that, you should use the _:ignore_exceptions_ option, like this:

```ruby
Whatever::Application.config.middleware.use ExceptionNotifier,
  :ignore_exceptions => ['ActionView::TemplateError'] + ExceptionNotifier.default_ignore_exceptions,
  :email => {
    :email_prefix         => "[Whatever] ",
    :sender_address       => %{"notifier" <notifier@example.com>},
    :exception_recipients => %w{exceptions@example.com}
  }
```

The above will make ExceptionNotifier ignore a *TemplateError*
exception, plus the ones ignored by default.
By default, ExceptionNotifier ignores _ActiveRecord::RecordNotFound_,
_AbstractController::ActionNotFound_ and
_ActionController::RountingError_.

* _:ignore_crawlers_

In some cases you may want to avoid getting notifications from exceptions
made by crawlers. Using _:ignore_crawlers_ option like this,

```ruby
Whatever::Application.config.middleware.use ExceptionNotifier,
  :ignore_crawlers => %w{Googlebot bingbot},
  :email => {
    :email_prefix         => "[Whatever] ",
    :sender_address       => %{"notifier" <notifier@example.com>},
    :exception_recipients => %w{exceptions@example.com}
  }
```

will prevent sending those unwanted notifications.

* _:ignore_if_

Last but not least, you can ignore exceptions based on a condition, by

```ruby
Whatever::Application.config.middleware.use ExceptionNotifier,
  :ignore_if => lambda { |env, e| e.message =~ /^Couldn't find Page with ID=/ },
  :email => {
    :email_prefix         => "[Whatever] ",
    :sender_address       => %{"notifier" <notifier@example.com>},
    :exception_recipients => %w{exceptions@example.com},
  }
```

You can make use of both the environment and the exception inside the lambda to decide wether to
avoid or not sending the notification.

### Headers

Additionally, you may want to set customized headers on the outcoming
emails. To do so, simply use the _:email_headers_ option:

```ruby
Whatever::Application.config.middleware.use ExceptionNotifier,
  :ignore_if => lambda { |env, e| e.message =~ /^Couldn't find Page with ID=/ },
  :email => {
    :email_prefix         => "[Whatever] ",
    :sender_address       => %{"notifier" <notifier@example.com>},
    :exception_recipients => %w{exceptions@example.com},
    :email_headers        => { "X-Custom-Header" => "foobar" }
  }
```

### Verbose

You can also choose to exclude the exception message from the subject, which is included by default.
Use _:verbose_subject => false_ to exclude it.

### Normalize subject

You can also choose to remove numbers from subject so they thread as a single one.
This is disabled by default.
Use _:normalize_subject => true_ to enable it.

### HTML

You may want to send multipart notifications instead of just plain text, which ExceptionNotification sends by default.
You can do so by adding this to the configuration: _:email_format => :html_.


Background Notifications
---

If you want to send notifications from a background process like
DelayedJob, you should use the `notify_exception` method like this:

```ruby
begin
  some code...
rescue => e
  ExceptionNotifier.notify_exception(e)
end
```

You can include information about the background process that created
the error by including a data parameter:

```ruby
begin
  some code...
rescue => exception
  ExceptionNotifier.notify_exception(exception,
    :data => {:worker => worker.to_s, :queue => queue, :payload => payload})
end
```


Manually notify of exception
---

If your controller action manually handles an error, the notifier will never be
run. To manually notify of an error you can do something like the following:

```ruby
rescue_from Exception, :with => :server_error

def server_error(exception)
  # Whatever code that handles the exception

  ExceptionNotifier.notify_exception(exception,
    :env => request.env, :data => {:message => "was doing something wrong"})
end
```

Notification
---

After an exception notification has been delivered the rack environment variable
'exception_notifier.delivered' will be set to `true`.


Override SMTP settings
---

You can use specific SMTP settings for notifications:

```ruby
Whatever::Application.config.middleware.use ExceptionNotifier,
  :email => {
    :email_prefix         => "[Whatever] ",
    :sender_address       => %{"notifier" <notifier@example.com>},
    :exception_recipients => %w{exceptions@example.com},
    :smtp_settings => {
      :user_name => "bob",
      :password => "password",
    }
  }
```

Versions
---

NOTE: Master branch is currently set for v4.0.0

For v3.0.1, see this tag:

<a href="http://github.com/smartinez87/exception_notification/tree/v3.0.1">http://github.com/smartinez87/exception_notification/tree/v3.0.1</a>

For v3.0.0, see this tag:

<a href="http://github.com/smartinez87/exception_notification/tree/v3.0.0">http://github.com/smartinez87/exception_notification/tree/v3.0.0</a>

For v2.6.1, see this tag:

<a href="http://github.com/smartinez87/exception_notification/tree/v2.6.1">http://github.com/smartinez87/exception_notification/tree/v2.6.1</a>

For previous releases, visit:

<a href="https://github.com/smartinez87/exception_notification/tags">https://github.com/smartinez87/exception_notification/tags</a>

If you are running Rails 2.3 then see the branch for that:

<a href="http://github.com/smartinez87/exception_notification/tree/2-3-stable">http://github.com/smartinez87/exception_notification/tree/2-3-stable</a>

If you are running pre-rack Rails then see this tag:

<a href="http://github.com/smartinez87/exception_notification/tree/pre-2-3">http://github.com/smartinez87/exception_notification/tree/pre-2-3</a>

Support and tickets
---

Here's the list of [issues](https://github.com/smartinez87/exception_notification/issues) we're currently working on.

To contribute, please read first the [Contributing
Guide](https://github.com/smartinez87/exception_notification/blob/master/CONTRIBUTING.md).

Copyright (c) 2005 Jamis Buck, released under the MIT license: <a href="http://www.opensource.org/licenses/MIT">http://www.opensource.org/licenses/MIT</a>
