title: Formatting
type: guide
order: 52
version: 0.9
---

JR by default uses some simple rules to format (and unformat) an attribute for (de-)serialization. Strings and Integers are output to JSON as is, and all other values have `.to_s` applied to them. This outputs something in all cases, but it is certainly not correct for every situation.

If you want to change the way an attribute is (de-)serialized, you have a couple of ways. The simplest method is to create a getter (and setter) method on the resource that overrides the attribute and applies the (un-)formatting there. For example:

```ruby
class PersonResource < JSONAPI::Resource
  attributes :name, :email, :last_login_time

  # Setter example
  def email=(new_email)
    @model.email = new_email.downcase
  end

  # Getter example
  def last_login_time
    @model.last_login_time.in_time_zone(@context[:current_user].time_zone).to_s
  end
end
```

This is simple to implement for a one-off situation. But in this example, it wouldn't be good if you wanted to apply the same formatting rules to all DateTime fields in your system. Another issue is the attribute on the resource will always return a formatted response, whether you want it or not.

## Value Formatters

To overcome the above limitations, JR uses Value Formatters. Value Formatters allow you to control the way values are handled for an attribute. The `format` can be set per attribute as it is declared in the resource. For example:

```ruby
class PersonResource < JSONAPI::Resource
  attributes :name, :email, :spoken_languages
  attribute :last_login_time, format: :date_with_utc_timezone

  # Getter/Setter for spoken_languages ...
end
```

A Value formatter has a `format` and an `unformat` method. Here's the base ValueFormatter and DefaultValueFormatter for reference:

```ruby
module JSONAPI
  class ValueFormatter < Formatter
    class << self
      def format(raw_value)
        super(raw_value)
      end

      def unformat(value)
        super(value)
      end
      ...
    end
  end
end

class DefaultValueFormatter < JSONAPI::ValueFormatter
  class << self
    def format(raw_value)
      case raw_value
        when Date, Time, DateTime, ActiveSupport::TimeWithZone, BigDecimal
          # Use the as_json methods added to various base classes by ActiveSupport
          return raw_value.as_json
        else
          return raw_value
      end
    end
  end
end
```

You can also create your own Value Formatter. Value Formatters must be named with the `format` name followed by `ValueFormatter`, i.e. `DateWithUTCTimezoneValueFormatter`, and derive from `JSONAPI::ValueFormatter`. It is recommended that you create a directory for your formatters, called `formatters`.

The `format` method is called by the `ResourceSerializer` as it is serializing a resource. The `format` method takes the `raw_value` parameter. `raw_value` is the value as read from the model.

The `unformat` method is called when processing the request. Each incoming attribute (except `links`) are run through the `unformat` method. The `unformat` method takes a `value`, which is the value as it comes in on the request. This allows you process the incoming value to alter its state before it is stored in the model.

### Use a Different Default Value Formatter

Another way to handle formatting is to set a different default value formatter. This will affect all attributes that do not have a `format` set. You can do this by overriding the `default_attribute_options` method for a resource (or a base resource for a system wide change).

```ruby
  def self.default_attribute_options
    {format: :my_default}
  end
```

and

```ruby
class MyDefaultValueFormatter < DefaultValueFormatter
  class << self
    def format(raw_value)
      case raw_value
        when DateTime
          return super(raw_value.in_time_zone('UTC'))
        else
          return super
      end
    end
  end
end
```

This way all DateTime values will be formatted to display in the UTC timezone.

## Key Format

By default, JR uses dasherized keys as per the [JSON API naming recommendations](http://jsonapi.org/recommendations/#naming).  This can be changed by specifying a different key formatter.

For example, to use camel cased keys with an initial lowercase character (JSON's default), create an initializer and add the following:

```ruby
JSONAPI.configure do |config|
  # built in key format options are :underscored_key, :camelized_key and :dasherized_key
  config.json_key_format = :camelized_key
end
```

This will cause the serializer to use the `CamelizedKeyFormatter`. You can also create your own `KeyFormatter`, for example:

```ruby
class UpperCamelizedKeyFormatter < JSONAPI::KeyFormatter
  class << self
    def format(key)
      super.camelize(:upper)
    end
  end
end
```

You would specify this in `JSONAPI.configure` as `:upper_camelized`.
