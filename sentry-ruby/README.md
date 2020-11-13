<p align="center">
  <a href="https://sentry.io" target="_blank" align="center">
    <img src="https://sentry-brand.storage.googleapis.com/sentry-logo-black.png" width="280">
  </a>
  <br>
</p>

# sentry-ruby, the Ruby Client for Sentry

---


[![Gem Version](https://img.shields.io/gem/v/sentry-ruby.svg)](https://rubygems.org/gems/sentry-ruby)
![Build Status](https://github.com/getsentry/sentry-ruby/workflows/sentry-ruby%20Test/badge.svg)
[![Coverage Status](https://img.shields.io/codecov/c/github/getsentry/sentry-ruby/master?logo=codecov)](https://codecov.io/gh/getsentry/sentry-ruby/branch/master)
[![Gem](https://img.shields.io/gem/dt/sentry-ruby.svg)](https://rubygems.org/gems/sentry-ruby/)
[![SemVer](https://api.dependabot.com/badges/compatibility_score?dependency-name=sentry-ruby&package-manager=bundler&version-scheme=semver)](https://dependabot.com/compatibility-score.html?dependency-name=sentry-ruby&package-manager=bundler&version-scheme=semver)


[Documentation](https://docs.sentry.io/clients/ruby/) | [Bug Tracker](https://github.com/getsentry/sentry-ruby/issues) | [Forum](https://forum.sentry.io/) | IRC: irc.freenode.net, #sentry

The official Ruby-language client and integration layer for the [Sentry](https://github.com/getsentry/sentry) error reporting API.


## Requirements

We test on Ruby 2.4, 2.5, 2.6 and 2.7 at the latest patchlevel/teeny version. We also support JRuby 9.0.

## Getting Started

### Install

```ruby
gem "sentry-ruby"
```

and depends on the integrations you want to have, you might also want to install these:

```ruby
gem "sentry-rails"
gem "sentry-sidekiq"
# and mores to come in the future!
```

### Sentry only runs when Sentry DSN is set

Sentry will capture and send exceptions to the Sentry server whenever its DSN is set. This makes environment-based configuration easy - if you don't want to send errors in a certain environment, just don't set the DSN in that environment!

```bash
# Set your SENTRY_DSN environment variable.
export SENTRY_DSN=http://public@example.com/project-id
```
```ruby
# Or you can configure the client in the code.
Sentry.init do |config|
  config.dsn = 'http://public@example.com/project-id'
end
```

### Sentry doesn't report some kinds of data by default

**Sentry ignores some exceptions by default** - most of these are related to 404s parameter parsing errors. [For a complete list, see the `IGNORE_DEFAULT` constant](https://github.com/getsentry/sentry-ruby/blob/master/sentry-ruby/lib/sentry/configuration.rb#L118) and the integration gems' `IGNORE_DEFAULT`, like [`sentry-rails`'s](https://github.com/getsentry/sentry-ruby/blob/update-readme/sentry-rails/lib/sentry/rails/configuration.rb#L12)

Sentry doesn't send personally identifiable information (pii) by default, such as request body, user ip or cookies. If you want those information to be sent, you can use the `send_default_pii` config option:

```ruby
Sentry.init do |config|
  # other configs
  config.send_default_pii = true
end
```

### Usage

`sentry-ruby` has a default integration with `Rack`, so you only need to use the middleware in your application like:

```
require 'rack'
require 'sentry'

use Sentry::Rack::CaptureException

run theapp
```

Otherwise, Sentry you can always use the capture helpers manually

```ruby
Sentry.capture_message("hello world!")

begin
  1 / 0
rescue ZeroDivisionError => exception
  Sentry.capture_exception(exception)
end
```

We also provide integrations with popular frameworks/libraries with the related extensions:

- [sentry-rails](https://github.com/getsentry/sentry-ruby/tree/master/sentry-rails)
- [sentry-sidekiq](https://github.com/getsentry/sentry-ruby/tree/master/sentry-sidekiq)

### More configuration

You're all set - but there's a few more settings you may want to know about too!

#### async

When an error or message occurs, the notification is immediately sent to Sentry. Sentry can be configured to send asynchronously:

```ruby
config.async = lambda { |event|
  Thread.new { Sentry.send_event(event) }
}
```

Using a thread to send events will be adequate for truly parallel Ruby platforms such as JRuby, though the benefit on MRI/CRuby will be limited. If the async callback raises an exception, Sentry will attempt to send synchronously.

Note that the naive example implementation has a major drawback - it can create an infinite number of threads. We recommend creating a background job, using your background job processor, that will send Sentry notifications in the background.

```ruby
config.async = lambda { |event| SentryJob.perform_later(event) }

class SentryJob < ActiveJob::Base
  queue_as :default

  def perform(event)
    Sentry.send_event(event)
  end
end
```

#### Contexts

In sentry-ruby, every event will inherit their contextual data from the current scope. So you can enrich the event's data by configuring the current scope like:

```ruby
Sentry.configure_scope do |scope|
  scope.set_user(id: 1, email: "test@example.com")

  scope.set_tag(:tag, "foo")
  scope.set_tags(tag_1: "foo", tag_2: "bar")

  scope.set_extra(:order_number, 1234)
  scope.set_extras(order_number: 1234, tickets_count: 4)
end

Sentry.capture_exception(exception) # the event will carry all those information now
```

Or build up a temporary scope for local information:

```ruby
Sentry.configure_scope do |scope|
  scope.set_tags(tag_1: "foo")
end

Sentry.with_scope do |scope|
  scope.set_tags(tag_1: "bar", tag_2: "baz")

  Sentry.capture_message("message") # this event will have 2 tags: tag_1 => "bar" and tag_2 => "baz"
end

Sentry.capture_message("another message") # this event will have 1 tag: tag_1 => "foo"
```

Of course, you can always assign the information on a per-event basis:

```ruby
Sentry.capture_exception(exception, tags: {foo: "bar"})
```

## More Information

* [Documentation](https://docs.sentry.io/clients/ruby/)
* [Bug Tracker](https://github.com/getsentry/sentry-ruby/issues)
* [Forum](https://forum.sentry.io/)
- [Discord](https://discord.gg/ez5KZN7)