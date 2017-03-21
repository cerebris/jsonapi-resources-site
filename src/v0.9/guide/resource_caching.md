title: Resource Caching
type: guide
order: 80
version: 0.9
---

To improve the response time of GET requests, JR can cache the generated JSON fragments for Resources which are suitable. First, set `config.resource_cache` to an ActiveSupport cache store:

```ruby
JSONAPI.configure do |config|
  config.resource_cache = Rails.cache
end
```

Then, on each Resource you want to cache, call the `caching` method:

```ruby
class PostResource < JSONAPI::Resource
  caching
end
```

See the [caveats section below](#Caching-Caveats) for situations where you might not want to enable caching on particular Resources.

The Resource model must also have a field that is updated whenever any of the model's data changes. The default Rails timestamps handle this pretty well, and the default cache key field is `updated_at` for this reason. You can use an alternate field (which you are then responsible for updating) by calling the `cache_field` method:

```ruby
class Post < ActiveRecord::Base
  before_save do
    if self.change_counter.nil?
      self.change_counter = 1
    elsif self.changed?
      self.change_counter += 1
    end
  end

  after_touch do
    update_attribute(:change_counter, self.change_counter + 1)
  end
end

class PostResource < JSONAPI::Resource
  caching
  cache_field :change_counter
end
```

If context affects the content of the serialized result, you must define a class method `attribute_caching_context` on that Resource, which should return a different value for contexts that produce different results. In particular, if the `meta` or `fetchable_fields` methods, or any method providing the actual content of an attribute, changes depending on context, then you must provide `attribute_caching_context`. The actual value it returns isn't important, what matters is that the value must be different if any relevant part of the context is different.

```ruby
class PostResource < JSONAPI::Resource
  caching

  attributes :title, :body, :secret_field

  def fetchable_fields
    return super if context.user.superuser?
    return super - [:secret_field]
  end

  def meta
    if context.user.can_see_creation_dates?
      return { created: _model.created_at }
    else
      return {}
    end
  end

  def self.attribute_caching_context(context)
    return {
      admin: context.user.superuser?,
      creation_date_viewer: context.user.can_see_creation_dates?
    }
  end
end
```

## Caching Caveats

* Models for cached Resources must update a cache key field whenever their data changes. However, if you bypass Rails and e.g. alter the database row directly without changing the `updated_at` field, the cached entry for that resource will be inaccurate. Also, `updated_at` provides a narrow race condition window; if a resource is updated twice in the same second, it's possible that only the first update will be cached. If you're concerned about this, you will need to find a way to make sure your models' cache fields change on every update, e.g. by using a unique random value or a monotonic clock.
* If an attribute's value is affected by related resources, e.g. the `spoken_languages` example above, then changes to the related resource must also touch the cache field on the resource that uses it. The `belongs_to` relation in ActiveRecord provides a `:touch` option for this purpose.
* JR does not actively clean the cache, so you must use an ActiveSupport cache that automatically expires old entries, or you will leak resources. The MemoryCache built in to Rails does this by default, but other caches will have to be configured with an `:expires_in` option and/or a cache-specific clearing mechanism.
* Similarly, if you make a substantial code change that affects a lot of serialized representations (i.e. changing the way an attribute is shown), you'll have to clear out all relevant cache entries yourself. The simplest way to do this is to run `JSONAPI.configuration.resource_cache.clear` from the console. You do not have to do this after merely adding or removing attributes; only changes that affect the actual content of attributes require manual cache clearing.
* If resource caching is enabled at all, then custom relationship methods on any resource might not always be used, even resources that are not cached. For example, if you manually define a `comments` method or `records_for_comments` method on a Resource that `has_many :comments`, you cannot expect it to be used when caching is enabled, even if you never call `caching` on that particular Resource. Instead, you should use relationship name lambdas.
* The above also applies to custom `find` or `find_by_key` methods. Instead, if you are using resource caching anywhere in your app, try overriding the `find_records` method to return an appropriate `ActiveRecord::Relation`.
* Caching relies on ActiveRecord features; you cannot enable caching on resources based on non-AR models, e.g. PORO objects or singleton resources.
* If you write a custom `ResourceSerializer` which takes new options, then you must define `config_description` to include those options if they might impact the serialized value:

```ruby
class MySerializer < JSONAPI::ResourceSerializer
  def initialize(primary_resource_klass, options = {})
    @my_special_option = options.delete(:my_special_option)
    super
  end

  def config_description(resource_klass)
    super.merge({my_special_option: @my_special_option})
  end
end
```
