# Ruby/Rack Middleware for Castle

[![Build Status](https://travis-ci.org/castle/castle-ruby-middleware.svg?branch=master)](https://travis-ci.org/castle/castle-ruby-middleware)
[![Coverage Status](https://coveralls.io/repos/github/castle/castle-ruby-middleware/badge.svg)](https://coveralls.io/github/castle/castle-ruby-middleware)
[![Maintainability](https://api.codeclimate.com/v1/badges/f5473d28967df1edf3b7/maintainability)](https://codeclimate.com/github/castle/castle-ruby-middleware/maintainability)

**Protect your users from stolen credentials. [Castle](https://castle.io) detects and mitigates account takeovers in web and mobile apps**

## Installation

Add this line to your Rack application's Gemfile:

```ruby
gem 'castle-middleware', git: 'https://github.com/castle/castle-ruby-middleware.git'
```

And set your Castle credentials in an initializer:

```ruby
Castle::Middleware.configure do |config|
  config.app_id = '186593875646714'
  config.api_secret = 'abcdefg123456789'
end
```

The middleware will insert itself into the Rack middleware stack.

## Usage

By default the middleware will insert [Castle.js](https://castle.io/docs/tracking) into the HEAD tag on all your pages.

### Mapping events

To start tracking events to Castle, you need to setup which routes should be mapped to
which Castle [security event](https://castle.io/docs/events). For authenticate endpoint use `authenticate: true` flag. for specifying referer use `referer` prop.

```yaml
# config/castle.yml
---
events:
  $login.failed:
    path: /session
    method: POST
    status: 401,
    properties:
      email: 'session.email' # Send user email extracted from params['session']['email']
  $login.succeeded: # Remember to register the current user, see below
    authenticate: true
    referer: /login
    path: /session
    method: POST
    status: 302
  $password_change.succeeded:
    path: !ruby/regexp '\/users\/\d+\/account'
    method: POST
    status: 200
```


By default the middleware will look for a YAML configuration file located at `config/castle.yml`. If you need to place this elsewhere this can be configured using the
`file_path` option, eg.:

```ruby
Castle::Middleware.configure do |config|
  config.file_path = './castle_config.yml'
end
```

#### Multiple paths per event

In some cases you might have multiple paths that should track the same event. Eg. if you have one login endpoint for API devices, and one for webapps. This is possible by specifying multiple paths as an array under each event definition:

```yaml
events:
  $login.succeeded:
    status: '302'
    method: POST
    path: "/users/sign_in"
    properties:
      email: user.email
  $login.failed:
    - path: /session
      method: POST
      status: 401
      properties:
        email: 'session.email'
    - path: /other_session
      method: PUT
      status: 300
      properties:
        email: 'session.email'
```


#### Devise

If you're using [Devise](https://github.com/plataformatec/devise) as authentication solution, check out [this example config](examples/castle_devise.yml) for how to map the most common security events


### Identifying the logged in user

Add `identify` config to configuration file. It should consist of nad id and user traits. Additionally you need to provide service in configuration where you provide user resource instance from a request data

```yaml

identify:
  id: uuid
  traits:
    email: email
    name: full_name
 
```

```ruby
Castle::Middleware.configure do |config|
  config.services.provide_user do |request|
    #User.find_by(id: request.session[:user_id])
    #provide user resource object from the request
  end
end

## Configuration

### inserting middleware in Rails

require 'castle/middleware/railties' # in you application.rb file or manually add in config/application.rb

```ruby
# config/application.rb
app.config.middleware.insert_after ActionDispatch::Flash,
                                    Castle::Middleware::Sensor
app.config.middleware.insert_after ActionDispatch::Flash,
                                    Castle::Middleware::Tracking
app.config.middleware.insert_after ActionDispatch::Flash,
                                    Castle::Middleware::Authenticating
```


### Async tracking

By default Castle sends tracking requests synchronously. To eg. use Sidekiq
to send requests in a background worker you can override the transport method

```ruby
# castle_worker.rb

class CastleWorker
  include Sidekiq::Worker

  def perform(params, context)
    ::Castle::Middleware.track(params, context)
  end
end

# initializers/castle.rb

Castle::Middleware.configure do |config|
  config.transport = lambda do |params, context|
    CastleWorker.perform_async(params, context)
  end
end

```

### Handling errors

By default, all Castle exceptions are handled silently, but you can also create a custom error handler:

```ruby
config.error_handler = lambda do |exception|
  # Handle error from Castle
end
```

## License

The gem is available as open source under the terms of the [MIT License](http://opensource.org/licenses/MIT).


