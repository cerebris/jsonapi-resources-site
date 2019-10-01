title: Introduction
type: guide
order: 0
version: 0.10
---

`JSONAPI::Resources`, or "JR", provides a framework for developing a server that complies with the [JSON API](http://jsonapi.org/) specification.

Like JSON API itself, JR's design is focused on the resources served by an API. JR needs little more than a definition of your resources, including their attributes and relationships, to make your server compliant with JSON API.

JR is designed to work with Rails 4.2+, and provides custom routes, controllers, and serializers. JR's resources may be backed by ActiveRecord models or by custom objects.

## Client Libraries

JSON API maintains a (non-verified) listing of [client libraries](http://jsonapi.org/implementations/#client-libraries)
which *should* be compatible with JSON API compliant server implementations such as JR.

## Installation

Add JR to your application's `Gemfile`:

```ruby
  gem 'jsonapi-resources'
```

And then execute:

```bash
bundle
```

Or install it yourself as:

```bash
gem install jsonapi-resources
```
