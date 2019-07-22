title: Configuration
type: guide
order: 90
version: 0.10
---

JR has a few configuration options, though the default settings will work well for most projects. To set configuration options create an initializer and add the options you wish to set. All options have defaults, so you only need to set the options that are different.

## Example

You wish to change the way multi word JSON keys are serialized. By default JR uses the JSON API recommendation for dasherizing (`foo-bar`) compound words.

### Initializer

Create a new file named `jsonapi-resources.rb` in your project's `config/initializers` directory and add the following:

```ruby
JSONAPI.configure do |config|

end
```

### Override the defaults

Now override the `json_key_format` and `route_format` to camelize (`fooBar`) compound words:

```ruby
JSONAPI.configure do |config|
  #:underscored_key, :camelized_key, :dasherized_key, or custom
  config.json_key_format = :camelized_key

  #:underscored_route, :camelized_route, :dasherized_route, or custom
  config.route_format = :camelized_route
end
```

See [formatters](formatting.html) for more details on formatters.

## Defaults

The default configuration options are shown below:

```ruby
JSONAPI.configure do |config|
  #:underscored_key, :camelized_key, :dasherized_key, or custom
  config.json_key_format = :dasherized_key

  #:underscored_route, :camelized_route, :dasherized_route, or custom
  config.route_format = :dasherized_route

  #:integer, :uuid, :string, or custom (provide a proc)
  config.resource_key_type = :integer

  # optional request features
  config.default_allow_include_to_one = true
  config.default_allow_include_to_many = true
  config.allow_sort = true
  config.allow_filter = true

  config.raise_if_parameters_not_allowed = true

  config.warn_on_route_setup_issues = true
  config.warn_on_missing_routes = true
  config.warn_on_performance_issues = true
  
  # :none, :offset, :paged, or a custom paginator name
  config.default_paginator = :none

  # Output pagination links at top level
  config.top_level_links_include_pagination = true

  config.default_page_size = 10
  config.maximum_page_size = 20

  # Metadata
  # Output record count in top level meta for find operation
  config.top_level_meta_include_record_count = false
  config.top_level_meta_record_count_key = :record_count

  config.top_level_meta_include_page_count = false
  config.top_level_meta_page_count_key = :page_count

  config.use_text_errors = false

  # Whether or not to include exception backtraces in JSONAPI error
  # responses.  Defaults to `false` in anything other than development or test.
  config.include_backtraces_in_errors = (Rails.env.development? || Rails.env.test?)

  # Whether or not to include exception application backtraces in JSONAPI error
  # responses.  Defaults to `false` in anything other than development or test.
  config.include_application_backtraces_in_errors = (Rails.env.development? || Rails.env.test?)

  # List of classes that should not be rescued by the operations processor.
  # For example, if you use Pundit for authorization, you might
  # raise a Pundit::NotAuthorizedError at some point during operations
  # processing. If you want to use Rails' `rescue_from` macro to
  # catch this error and render a 403 status code, you should add
  # the `Pundit::NotAuthorizedError` to the `exception_class_whitelist`.
  config.exception_class_whitelist = []

  # Resource Linkage
  # Controls the serialization of resource linkage for non compound documents
  # NOTE: always_include_to_many_linkage_data is not currently implemented
  config.always_include_to_one_linkage_data = false
  config.always_include_to_many_linkage_data = false

  # The default Operation Processor to use if one is not defined specifically
  # for a Resource.
  config.default_processor_klass = JSONAPI::Processor

  # Allows transactions for creating and updating records
  # Set this to false if your backend does not support transactions (e.g. Mongodb)
  config.allow_transactions = true

  # Formatter Caching
  # Set to false to disable caching of string operations on keys and links.
  # Note that unlike the resource cache, formatter caching is always done
  # internally in-memory and per-thread; no ActiveSupport::Cache is used.
  config.cache_formatters = true

  # Relationship reflection invokes the related resource when updates
  # are made to a has_many relationship. By default relationship_reflection
  # is turned off because it imposes a small performance penalty.
  config.use_relationship_reflection = false

  # Resource cache
  # An ActiveSupport::Cache::Store or similar, used by Resources with caching enabled.
  # Set to `nil` (the default) to disable caching, or to `Rails.cache` to use the
  # Rails cache store.
  config.resource_cache = nil

  # Default cache resources
  # Cache resources by default. Individual resources can be excluded from caching by calling:
  # `caching false`
  config.default_caching = false

  # Default resource cache field
  # On Resources with caching enabled, this field will be used to check for out-of-date
  # cache entries, unless overridden on a specific Resource. Defaults to "updated_at".
  config.default_resource_cache_field = :updated_at

  # Resource cache digest function
  # Provide a callable that returns a unique value for string inputs with
  # low chance of collision. The default is SHA256 base64.
  config.resource_cache_digest_function = Digest::SHA2.new.method(:base64digest)

  # Resource cache usage reporting
  # Optionally provide a callable which JSONAPI will call with information about cache
  # performance. Should accept three arguments: resource name, hits count, misses count.
  config.resource_cache_usage_report_function = nil
end
```

Note: If using custom classes (such as a `CustomPaginator`), be sure to require them at the top of the initializer before usage.
