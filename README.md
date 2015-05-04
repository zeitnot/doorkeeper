# Doorkeeper - awesome oauth provider for your Rails app.

[![Build Status](https://travis-ci.org/doorkeeper-gem/doorkeeper.svg?branch=master)](https://travis-ci.org/doorkeeper-gem/doorkeeper)
[![Dependency Status](https://gemnasium.com/applicake/doorkeeper.svg?travis)](https://gemnasium.com/applicake/doorkeeper)
[![Code Climate](https://codeclimate.com/github/applicake/doorkeeper.svg)](https://codeclimate.com/github/applicake/doorkeeper)
[![Gem Version](https://badge.fury.io/rb/doorkeeper.svg)](https://rubygems.org/gems/doorkeeper)

Doorkeeper is a gem that makes it easy to introduce OAuth 2 provider
functionality to your Rails or Grape application.

[PR 567]: https://github.com/doorkeeper-gem/doorkeeper/pull/567


## Documentation valid for `master` branch

Please check the documentation for the version of doorkeeper you are using in:
https://github.com/doorkeeper-gem/doorkeeper/releases.

## Table of Contents

- [Useful links](#useful-links)
- [Installation](#installation)
- [Configuration](#configuration)
    - [Active Record](#active-record)
    - [Other ORMs](#other-orms)
    - [Routes](#routes)
    - [Authenticating](#authenticating)
    - [Internationalization (I18n)](#internationalization-i18n)
- [Protecting resources with OAuth (a.k.a your API endpoint)](#protecting-resources-with-oauth-aka-your-api-endpoint)
    - [Protect your API with OAuth when using Grape](#protect-your-api-with-oauth-when-using-grape)
    - [Route Constraints and other integrations](#route-constraints-and-other-integrations)
    - [Access Token Scopes](#access-token-scopes)
    - [Custom Access Token Generator](#custom-access-token-generator)
    - [Authenticated resource owner](#authenticated-resource-owner)
    - [Applications list](#applications-list)
- [Other customizations](#other-customizations)
- [Upgrading](#upgrading)
- [Development](#development)
- [Contributing](#contributing)
- [Other resources](#other-resources)
    - [Wiki](#wiki)
    - [Live demo](#live-demo)
    - [Screencast](#screencast)
    - [Client applications](#client-applications)
    - [Contributors](#contributors)
    - [IETF Standards](#ietf-standards)
    - [License](#license)


## Useful links

- For documentation, please check out our [wiki](https://github.com/doorkeeper-gem/doorkeeper/wiki)
- For general questions, please post it in [stack overflow](http://stackoverflow.com/questions/tagged/doorkeeper)

## Installation

Put this in your Gemfile:

``` ruby
gem 'doorkeeper'
```

Run the installation generator with:

    rails generate doorkeeper:install

This will install the doorkeeper initializer into `config/initializers/doorkeeper.rb`.

## Configuration

### Active Record

By default doorkeeper is configured to use active record, so to start you have
to generate the migration tables:

    rails generate doorkeeper:migration

Don't forget to run the migration with:

    rake db:migrate

### Other ORMs

See [doorkeeper-orms project] for mongoid and mongomapper support. Follow along
the implementation in that repository to extend doorkeeper with other ORMs.

[doorkeeper-orms project]: https://github.com/doorkeeper-gem/doorkeeper-orms

### Routes

The installation script will also automatically add the Doorkeeper routes into
your app, like this:

``` ruby
Rails.application.routes.draw do
  use_doorkeeper
  # your routes
end
```

This will mount following routes:

    GET       /oauth/authorize/:code
    GET       /oauth/authorize
    POST      /oauth/authorize
    PUT       /oauth/authorize
    DELETE    /oauth/authorize
    POST      /oauth/token
    POST      /oauth/revoke
    resources /oauth/applications
    GET       /oauth/authorized_applications
    DELETE    /oauth/authorized_applications/:id
    GET       /oauth/token/info

For more information on how to customize routes, check out [this page on the
wiki](https://github.com/doorkeeper-gem/doorkeeper/wiki/Customizing-routes).

### Authenticating

You need to configure Doorkeeper in order to provide `resource_owner` model
and authentication block `initializers/doorkeeper.rb`

``` ruby
Doorkeeper.configure do
  resource_owner_authenticator do
    User.find_by_id(session[:current_user_id]) || redirect_to(login_url)
  end
end
```

This code is run in the context of your application so you have access to your
models, session or routes helpers. However, since this code is not run in the
context of your application's `ApplicationController` it doesn't have access to
the methods defined over there.

You may want to check other ways of authentication
[here](https://github.com/doorkeeper-gem/doorkeeper/wiki/Authenticating-using-Clearance-or-DIY).


### Internationalization (I18n)

See language files in [the I18n repository](https://github.com/doorkeeper-gem/doorkeeper-i18n).


## Protecting resources with OAuth (a.k.a your API endpoint)

To protect your API with OAuth, you just need to setup `before_action`s
specifying the actions you want to protect. For example:

``` ruby
class Api::V1::ProductsController < Api::V1::ApiController
  before_action :doorkeeper_authorize! # Require access token for all actions

  # your actions
end
```

You can pass any option `before_action` accepts, such as `if`, `only`,
`except`, and others.

### Protect your API with OAuth when using Grape

As of [PR 567] doorkeeper has helpers for Grape. One of them is
`doorkeeper_authorize!` and can be used in a similar way as an example above.
Note that you have to use `require 'doorkeeper/grape/helpers'` and
`helpers Doorkeeper::Grape::Helpers`.

For more information about integration with Grape see the [Wiki].

[PR 567]: https://github.com/doorkeeper-gem/doorkeeper/pull/567
[Wiki]: https://github.com/doorkeeper-gem/doorkeeper/wiki/Grape-Integration

``` ruby
require 'doorkeeper/grape/helpers'

module API
  module V1
    class Users < Grape::API
      helpers Doorkeeper::Grape::Helpers

      before do
        doorkeeper_authorize!
      end

      # ...
    end
  end
end
```


### Route Constraints and other integrations

You can leverage the `Doorkeeper.authenticate` facade to easily extract a
`Doorkeeper::OAuth::Token` based on the current request. You can then ensure
that token is still good, find its associated `#resource_owner_id`, etc.

```ruby
module Constraint
  class Authenticated

    def matches?(request)
      token = Doorkeeper.authenticate(request)
      token && token.accessible?
    end

  end
end
```

For more information about integration and other integrations, check out [the
related wiki
page](https://github.com/doorkeeper-gem/doorkeeper/wiki/ActionController::Metal-with-doorkeeper).

### Access Token Scopes

You can also require the access token to have specific scopes in certain
actions:

First configure the scopes in `initializers/doorkeeper.rb`

```ruby
Doorkeeper.configure do
  default_scopes :public # if no scope was requested, this will be the default
  optional_scopes :admin, :write
end
```

And in your controllers:

```ruby
class Api::V1::ProductsController < Api::V1::ApiController
  before_action -> { doorkeeper_authorize! :public }, only: :index
  before_action only: [:create, :update, :destroy] do
    doorkeeper_authorize! :admin, :write
  end
end
```

Please note that there is a logical OR between multiple required scopes. In
above example, `doorkeeper_authorize! :admin, :write` means that the access
token is required to have either `:admin` scope or `:write` scope, but not need
have both of them.

If want to require the access token to have multiple scopes at the same time,
use multiple `doorkeeper_authorize!`, for example:

```ruby
class Api::V1::ProductsController < Api::V1::ApiController
  before_action -> { doorkeeper_authorize! :public }, only: :index
  before_action only: [:create, :update, :destroy] do
    doorkeeper_authorize! :admin
    doorkeeper_authorize! :write
  end
end
```

In above example, a client can call `:create` action only if its access token
have both `:admin` and `:write` scopes.

### Custom Access Token Generator

By default a 32 bit access token will be generated. If you require a custom
token, such as [JWT](http://jwt.io), specify an object that responds to
`.generate(options = {})` and returns a string to be used as the token.

```ruby
Doorkeeper.configure do
  access_token_generator "Doorkeeper::JWT"
end
```

JWT token support is available with
[Doorkeeper-JWT](https://github.com/chriswarren/doorkeeper-jwt).


### Authenticated resource owner

If you want to return data based on the current resource owner, in other
words, the access token owner, you may want to define a method in your
controller that returns the resource owner instance:

``` ruby
class Api::V1::CredentialsController < Api::V1::ApiController
  before_action :doorkeeper_authorize!
  respond_to    :json

  # GET /me.json
  def me
    respond_with current_resource_owner
  end

  private

  # Find the user that owns the access token
  def current_resource_owner
    User.find(doorkeeper_token.resource_owner_id) if doorkeeper_token
  end
end
```

In this example, we're returning the credentials (`me.json`) of the access
token owner.

### Applications list

By default, the applications list (`/oauth/applications`) is public available.
To protect the endpoint you should uncomment these lines:

```ruby
# config/initializers/doorkeeper.rb
Doorkeeper.configure do
  admin_authenticator do |routes|
    Admin.find_by_id(session[:admin_id]) || redirect_to(routes.new_admin_session_url)
  end
end
```

The logic is the same as the `resource_owner_authenticator` block. **Note:**
since the application list is just a scaffold, it's recommended to either
customize the controller used by the list or skip the controller at all. For
more information see the page [in the
wiki](https://github.com/doorkeeper-gem/doorkeeper/wiki/Customizing-routes).

## Other customizations

- [Associate users to OAuth applications (ownership)](https://github.com/doorkeeper-gem/doorkeeper/wiki/Associate-users-to-OAuth-applications-%28ownership%29)
- [CORS - Cross Origin Resource Sharing](https://github.com/doorkeeper-gem/doorkeeper/wiki/%5BCORS%5D-Cross-Origin-Resource-Sharing)

## Upgrading

If you want to upgrade doorkeeper to a new version, check out the [upgrading
notes](https://github.com/doorkeeper-gem/doorkeeper/wiki/Migration-from-old-versions)
and take a look at the
[changelog](https://github.com/doorkeeper-gem/doorkeeper/blob/master/CHANGELOG.md).

## Development

To run the local engine server:

```
bundle install
bundle exec rails server
````

By default, it uses the latest Rails version with ActiveRecord. To run the
tests with a specific ORM and Rails version:

```
rails=4.2.0 orm=active_record bundle exec rake
```

Or you might prefer to run `script/run_all` to integrate against all ORMs.

## Contributing

Want to contribute and don't know where to start? Check out [features we're
missing](https://github.com/doorkeeper-gem/doorkeeper/wiki/Supported-Features),
create [example
apps](https://github.com/doorkeeper-gem/doorkeeper/wiki/Example-Applications),
integrate the gem with your app and let us know!

Also, check out our [contributing guidelines
page](https://github.com/doorkeeper-gem/doorkeeper/wiki/Contributing).

## Other resources

### Wiki

You can find everything about doorkeeper in our [wiki
here](https://github.com/doorkeeper-gem/doorkeeper/wiki).

### Live demo

Check out this [live demo](http://doorkeeper-provider.herokuapp.com) hosted on
heroku. For more demos check out [the
wiki](https://github.com/doorkeeper-gem/doorkeeper/wiki/Example-Applications).

### Screencast

Check out this screencast from [railscasts.com](http://railscasts.com/): [#353
OAuth with
Doorkeeper](http://railscasts.com/episodes/353-oauth-with-doorkeeper)

### Client applications

After you set up the provider, you may want to create a client application to
test the integration. Check out these [client
examples](https://github.com/doorkeeper-gem/doorkeeper/wiki/Example-Applications)
in our wiki or follow this [tutorial
here](https://github.com/doorkeeper-gem/doorkeeper/wiki/Testing-your-provider-with-OAuth2-gem).

### Contributors

Thanks to all our [awesome
contributors](https://github.com/doorkeeper-gem/doorkeeper/contributors)!


### IETF Standards

* [The OAuth 2.0 Authorization Framework](http://tools.ietf.org/html/rfc6749)
* [OAuth 2.0 Threat Model and Security Considerations](http://tools.ietf.org/html/rfc6819)

### License

MIT License. Copyright 2011 Applicake.
[http://applicake.com](http://applicake.com)
