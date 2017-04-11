title: Resource Caching
type: guide
order: 80
version: 0.10
---

To improve the response time of GET requests, JR can cache the generated JSON fragments for some or all of your Resources, using a [key-based cache expiration](https://signalvnoise.com/posts/3113-how-key-based-cache-expiration-works) system.

To begin, set `config.resource_cache` to an ActiveSupport cache store:

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

The Post model in this example must have the Rails timestamp field `updated_at`.

See the ["Resources to Not Cache" section below](#Resources-to-Not-Cache) for situations where you might not want to enable caching on particular Resources.

Also, when caching is enabled, please be careful about direct database manipulation. If you alter a database row without changing the `updated_at` field, the cached entry for that resource will be inaccurate.

## Cache Field

Instead of the default `updated_at`, you can use a different field (and take on responsibility for updating it) by calling the `cache_field` method:

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

One reason to do this is that `updated_at` provides a narrow race condition window. If a resource is updated twice in the same second, it's possible that only the first update will be cached. If you're concerned about this, you will need to find a way to make sure your models' cache fields change on every update, e.g. by using a unique random value or a monotonic clock.

## Cleaning the Cache

JR does **not** actively clean the cache, so you must use an ActiveSupport cache that automatically expires old entries, or you **will** leak resources. The default behavior of Rails' MemoryCache is good, but most other caches will have to be configured with an `:expires_in` option and/or a cache-specific clearing mechanism. In a Redis configuration for example, you will need to set `maxmemory` to a reasonably high size, and set `maxmemory-policy` to `allkeys-lru`.

Also, sometimes you may want to manually clear the cache. If you make a code change that affects serialized representations (i.e. changing the way an attribute is shown), or if you think that there might be invalid cache entries, you can clear the cache by running `JSONAPI.configuration.resource_cache.clear` from the console.

You do not have to manually clear the cache after merely adding or removing attributes on your Resource, because the field list is part of the cache key.

## Side-Effects of Enabling Resource Caching

When `JSONAPI.configuration.resource_cache` is set, JR changes the way it performs ActiveRecord queries on **all** show and index requests, even if a particular request doesn't involve any Resource classes that are cached.

In particular, certain methods you can define on your Resources will not reliably be called by JR operations when caching is enabled:

* Overridden `find`, `find_by_key`, and `records_for` methods will not reliably be called.
* Custom relationship methods will not reliably be called. For example, if you manually define a `comments` method or `records_for_comments` method on a Resource that `has_many :comments`, do not expect these to be called.

Instead of those, override the `records` class method on Resource; cache lookups will reliably access all records through the scope it returns. In particular, this is the correct way to implement view security; if you want to restrict visibility of a Resource depending on the current context, write a `records` class method for that Resource:

```ruby
class PostResource < JSONAPI::Resource
  caching

  attributes :title, :public
  has_one :owner

  def self.records(options = {})
    context = options[:context] || {}
    current_user = context[:current_user]

    if current_user
      Post.where('public = 1 OR owner_id = ?', current_user.id)
    else
      Post.where('public = 1')
    end
  end
end
```

## Resources to Not Cache

After setting `JSONAPI.configuration.resource_cache`, you may still choose to leave some Resources uncached for various reasons:

* If your Resource is not based on ActiveRecord, e.g. it uses PORO objects or singleton resources or a different ORM/ODM backend. Caching relies on ActiveRecord features.
* If a Resource's attributes depend on many things outside that Resource, e.g. flattened relationships, but it would be too cumbersome to have all those touch the parent resource on every change.
* If the content of attributes is affected by context in a way that is too difficult to handle with `attribute_caching_context`, as described below.

## Caching and Context

If context affects the output of any method providing the actual content of an attribute, or the `meta` or `fetchable_fields` methods, then you must provide a class method on your Resource named `attribute_caching_context`. This method should a subset of the context that is (a) serializable and (b) uniquely identifies the caching situation:

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

This is necessary because cache lookups do not create instances of your Resource, and therefore cannot call instance methods like `fetchable_fields`. Instead, they have to rely on finding the correct cached representation of the resource, the one generated when these methods were called with the correct context the first time. The attribute caching context is a way to let JR "sub-categorize" your cache by the various parts of your context that affect these instance methods. The same mechanism is also used internally by JR when clients request sparse fieldsets; a cached sparse representation and the cached representation with all attributes don't collide with each other.

This becomes trickier if you depend on the state of the model, not just on the state of the context. For example, suppose you have a `UserResource#fetchable_fields` that excludes `:email` unless the current user's id matches the resource's id. You would have to put the user's id in the attribute caching context, which means that every client would have their own independent cache of Users. This would be very inefficient, so probably it would be better to just leave UserResource uncached in this case.

## Custom Serializers

If you write a custom `ResourceSerializer` which takes new options, then you must define `config_description` to include those options if they might impact the serialized value:

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
