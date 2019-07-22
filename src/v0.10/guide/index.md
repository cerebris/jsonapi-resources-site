title: Introduction
type: guide
order: 0
version: 0.10
---

**Note: Version 0.10 is currently in Beta and should not be considered production ready.**

`JSONAPI::Resources`, or "JR", provides a framework for developing a server that complies with the [JSON API](http://jsonapi.org/) specification.

Like JSON API itself, JR's design is focused on the resources served by an API. JR needs little more than a definition of your resources, including their attributes and relationships, to make your server compliant with JSON API.

JR is designed to work with Rails 4.2+, and provides custom routes, controllers, and serializers. JR's resources may be backed by ActiveRecord models or by custom objects.

## Client Libraries

JSON API maintains a (non-verified) listing of [client libraries](http://jsonapi.org/implementations/#client-libraries)
which *should* be compatible with JSON API compliant server implementations such as JR.

**Note: Multiple operation support is a feature not yet supported by JSON API. As such there are currently no clients for this code.**

## Installation

Add JR to your application's `Gemfile`:

```ruby
  # For the latest release production level release
  # gem 'jsonapi-resources'

  # For the latest release preproduction level release (the version covered here)
  gem 'jsonapi-resources', '~>0.10.0.beta8'

  # Use alpha level code at your own risk
  # gem 'jsonapi-resources', git: 'https://github.com/cerebris/jsonapi-resources.git', branch: 'master'

```

And then execute:

```bash
bundle
```
