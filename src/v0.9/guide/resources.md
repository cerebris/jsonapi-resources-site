title: Resources
type: guide
order: 20
version: 0.9
---

Resources define the public interface to your API. A resource defines which attributes are exposed, as well as relationships to other resources.

Resource definitions should by convention be placed in a directory under app named resources, `app/resources`. The file name should be the single underscored name of the model that backs the resource with `_resource.rb` appended. For example, a `Contact` model's resource should have a class named `ContactResource` defined in a file named `contact_resource.rb`.

## JSONAPI::Resource

Resources must be derived from `JSONAPI::Resource`, or a class that is itself derived from `JSONAPI::Resource`.

For example:

```ruby
class ContactResource < JSONAPI::Resource
end
```

A jsonapi-resource generator is available
```bash
rails generate jsonapi:resource contact
```

### Abstract Resources

Resources that are not backed by a model (purely used as base classes for other resources) should be declared as abstract.

Because abstract resources do not expect to be backed by a model, they won't attempt to discover the model class or any of its relationships.

```ruby
class BaseResource < JSONAPI::Resource
  abstract

  has_one :creator
end

class ContactResource < BaseResource
end
```

### Immutable Resources

Resources that are immutable should be declared as such with the `immutable` method. Immutable resources will only generate routes for `index`, `show` and `show_relationship`.

#### Immutable for Readonly

Some resources are read-only and are not to be modified through the API. Declaring a resource as immutable prevents creation of routes that allow modification of the resource.

#### Immutable Heterogeneous Collections

Immutable resources can be used as the basis for a heterogeneous collection. Resources in heterogeneous collections can still be mutated through their own type-specific endpoints.

```ruby
class VehicleResource < JSONAPI::Resource
  immutable

  has_one :owner
  attributes :make, :model, :serial_number
end

class CarResource < VehicleResource
  attributes :drive_layout
  has_one :driver
end

class BoatResource < VehicleResource
  attributes :length_at_water_line
  has_one :captain
end

# routes
  jsonapi_resources :vehicles
  jsonapi_resources :cars
  jsonapi_resources :boats

```

In the above example vehicles are immutable. A call to `/vehicles` or `/vehicles/1` will return vehicles with types of either `car` or `boat`. But calls to PUT or POST a `car` must be made to `/cars`. The rails models backing the above code use Single Table Inheritance.

## Context

Sometimes you will want to access things such as the current logged in user (and other state only available within your controllers) from within your resource classes. To make this state available to a resource class you need to put it into the context hash - this can be done via a `context` method on one of your controllers or across all controllers using ApplicationController.

For example:

```ruby
class ApplicationController < JSONAPI::ResourceController
  def context
    {current_user: current_user}
  end
end

# Specific resource controllers derive from ApplicationController
# and share its context
class PeopleController < ApplicationController

end

# Assuming you don't permit user_id (so the client won't assign a wrong user to own the object)
# you can ensure the current user is assigned the record by using the controller's context hash.
class PeopleResource < JSONAPI::Resource
  before_save do
    @model.user_id = context[:current_user].id if @model.new_record?
  end
end
```

You can put things that affect serialization and resource configuration into the context.

## Attributes

Any of a resource's attributes that are accessible must be explicitly declared. Single attributes can be declared using the `attribute` method, and multiple attributes can be declared with the `attributes` method on the resource class.

For example:

```ruby
class ContactResource < JSONAPI::Resource
  attribute :name_first
  attributes :name_last, :email, :twitter
end
```

This resource has 4 defined attributes: `name_first`, `name_last`, `email`, `twitter`, as well as the automatically defined attributes `id` and `type`. By default these attributes must exist on the model that is handled by the resource.

A resource object wraps a Ruby object, usually an `ActiveModel` record, which is available as the `@model` variable. This allows a resource's methods to access the underlying model.

For example, a computed attribute for `full_name` could be defined as such:

```ruby
class ContactResource < JSONAPI::Resource
  attributes :name_first, :name_last, :email, :twitter
  attribute :full_name

  def full_name
    "#{@model.name_first}, #{@model.name_last}"
  end
end
```

### Attribute Delegation

Normally resource attributes map to an attribute on the model of the same name. Using the `delegate` option allows a resource attribute to map to a differently named model attribute. For example:

```ruby
class ContactResource < JSONAPI::Resource
  attribute :name_first, delegate: :first_name
  attribute :name_last, delegate: :last_name
end
```

### Fetchable Attributes

By default all attributes are assumed to be fetchable. The list of fetchable attributes can be filtered by overriding the `fetchable_fields` method.

Here's an example that prevents guest users from seeing the `email` field:

```ruby
class AuthorResource < JSONAPI::Resource
  attributes :name, :email
  model_name 'Person'
  has_many :posts

  def fetchable_fields
    if (context[:current_user].guest)
      super - [:email]
    else
      super
    end
  end
end
```

Context flows through from the controller to the resource and can be used to control the attributes based on the current user (or other value).

### Creatable and Updatable Attributes

By default all attributes are assumed to be updatable and creatable. To prevent some attributes from being accepted by the `update` or `create` methods, override the `self.updatable_fields` and `self.creatable_fields` methods on a resource.

This example prevents `full_name` from being set:

```ruby
class ContactResource < JSONAPI::Resource
  attributes :name_first, :name_last, :full_name

  def full_name
    "#{@model.name_first}, #{@model.name_last}"
  end

  def self.updatable_fields(context)
    super - [:full_name]
  end

  def self.creatable_fields(context)
    super - [:full_name]
  end
end
```

The `context` is not by default used by the `ResourceController`, but may be used if you override the controller methods. By using the context you have the option to determine the creatable and updatable fields based on the user.

### Sortable Attributes

JR supports [sorting primary resources by multiple sort criteria](http://jsonapi.org/format/#fetching-sorting).

By default all attributes are assumed to be sortable. To prevent some attributes from being sortable, override the `self.sortable_fields` method on a resource.

Here's an example that prevents sorting by post's `body`:

```ruby
class PostResource < JSONAPI::Resource
  attributes :title, :body

  def self.sortable_fields(context)
    super(context) - [:body]
  end
end
```

JR also supports sorting primary resources by fields on relationships.

Here's an example of sorting books by the author name:

```ruby
class Book < ActiveRecord::Base
  belongs_to :author
end

class Author < ActiveRecord::Base
  has_many :books
end

class BookResource < JSONAPI::Resource
  attributes :title, :body

  def self.sortable_fields(context)
    super + [:"author.name"]
  end
end
```
The request will look something like:
```
GET /books?include=author&sort=author.name
```

By default, a sort will happen in ascending order. In order to sort in descending order, just append a minus sign in front of the variable you are sorting by:
```
GET /books?include=author&sort=-author.name
```


#### Default sorting

By default JR sorts ascending on the `id` of the primary resource, unless the request specifies an alternate sort order. To override this you may override the `self.default_sort` on a `resource`. `default_sort` should return an array of `sort_param` hashes. A `sort_param` hash contains a `field` and a `direction`, with `direction` being either `:asc` or `:desc`.

For example:

```ruby
  def self.default_sort
    [{field: 'name_last', direction: :desc}, {field: 'name_first', direction: :desc}]
  end
```

### Attribute Formatting

Attributes can have a `Format`. By default all attributes use the default formatter. If an attribute has the `format` option set the system will attempt to find a formatter based on this name. In the following example the `last_login_time` will be returned formatted to a certain time zone:

```ruby
class PersonResource < JSONAPI::Resource
  attributes :name, :email
  attribute :last_login_time, format: :date_with_timezone
end
```

The system will lookup a value formatter named `DateWithTimezoneValueFormatter` and will use this when serializing and updating the attribute. See the [Value Formatters](#value-formatters) section for more details.

### Flattening a Rails relationship

It is possible to flatten Rails relationships into attributes by using getters and setters. This can become handy if a relation needs to be created alongside the creation of the main object which can be the case if there is a bi-directional presence validation. For example:

```ruby
# Given Models
class Person < ActiveRecord::Base
  has_many :spoken_languages
  validates :name, :email, :spoken_languages, presence: true
end

class SpokenLanguage < ActiveRecord::Base
  belongs_to :person, inverse_of: :spoken_languages
  validates :person, :language_code, presence: true
end

# Resource with getters and setter
class PersonResource < JSONAPI::Resource
  attributes :name, :email, :spoken_languages

  # Getter
  def spoken_languages
    @model.spoken_languages.pluck(:language_code)
  end

  # Setter (because spoken_languages needed for creation)
  def spoken_languages=(new_spoken_language_codes)
    @model.spoken_languages.destroy_all
    new_spoken_language_codes.each do |new_lang_code|
      @model.spoken_languages.build(language_code: new_lang_code)
    end
  end
end
```

## Primary Key

Resources are always represented using a key of `id`. The resource will interrogate the model to find the primary key. If the underlying model does not use `id` as the primary key _and_ does not support the `primary_key` method you must use the `primary_key` method to tell the resource which field on the model to use as the primary key. **Note:** this _must_ be the actual primary key of the model.

By default only integer values are allowed for primary key. To change this behavior you can set the `resource_key_type`
configuration option:

```ruby
JSONAPI.configure do |config|
  # Allowed values are :integer(default), :uuid, :string, or a proc
  config.resource_key_type = :uuid
end
```

### Override key type on a resource

You can override the default resource key type on a per-resource basis by calling `key_type` in the resource class, with the same allowed values as the `resource_key_type` configuration option.

```ruby
class ContactResource < JSONAPI::Resource
  attribute :id
  attributes :name_first, :name_last, :email, :twitter
  key_type :uuid
end
```

### Custom resource key validators

If you need more control over the key, you can override the #verify_key method on your resource, or set a lambda that accepts key and context arguments in `config/initializers/jsonapi_resources.rb`:

```ruby
JSONAPI.configure do |config|
  config.resource_key_type = -> (key, context) { key && String(key) }
end
```

## Model Name

The name of the underlying model is inferred from the Resource name. It can be overridden by use of the `model_name` method. For example:

```ruby
class AuthorResource < JSONAPI::Resource
  attribute :name
  model_name 'Person'
  has_many :posts
end
```

## Model Hints

Resource instances are created from model records. The determination of the correct resource type is performed using a simple rule based on the model's name. The name is used to find a resource in the same module (as the originating resource) that matches the name. This usually works quite well, however it can fail when model names do not match resource names. It can also fail when using namespaced models. In this case a `model_hint` can be created to map model names to resources. For example:

```ruby
class AuthorResource < JSONAPI::Resource
  attribute :name
  model_name 'Person'
  model_hint model: Commenter, resource: :special_person

  has_many :posts
  has_many :commenters
end
```

Note that when `model_name` is set a corresponding `model_hint` is also added. This can be skipped by using the `add_model_hint` option set to false. For example:

```ruby
class AuthorResource < JSONAPI::Resource
  model_name 'Legacy::Person', add_model_hint: false
end
```

Model hints inherit from parent resources, but are not global in scope. The `model_hint` method accepts `model` and `resource` named parameters. `model` takes an ActiveRecord class or class name (defaults to the model name), and `resource` takes a resource type or a resource class (defaults to the current resource's type).

## Relationships

Related resources need to be specified in the resource. These may be declared with the `relationship` or the `has_one` and the `has_many` methods.

Here's a simple example using the `relationship` method where a post has a single author and an author can have many posts:

```ruby
class PostResource < JSONAPI::Resource
  attributes :title, :body

  relationship :author, to: :one
end
```

And the corresponding author:

```ruby
class AuthorResource < JSONAPI::Resource
  attribute :name

  relationship :posts, to: :many
end
```

And here's the equivalent resources using the `has_one` and `has_many` methods:

```ruby
class PostResource < JSONAPI::Resource
  attributes :title, :body

  has_one :author
end
```

And the corresponding author:

```ruby
class AuthorResource < JSONAPI::Resource
  attribute :name

  has_many :posts
end
```

### Options

The relationship methods (`relationship`, `has_one`, and `has_many`) support the following options:

 * `class_name` - a string specifying the underlying class for the related resource. Defaults to the `class_name` property on the underlying model.
 * `foreign_key` - the method on the resource used to fetch the related resource. Defaults to `<resource_name>_id` for has_one and `<resource_name>_ids` for has_many relationships.
 * `acts_as_set` - allows the entire set of related records to be replaced in one operation. Defaults to false if not set.
 * `polymorphic` - set to true to identify relationships that are polymorphic.
 * `relation_name` - the name of the relation to use on the model. A lambda may be provided which allows conditional selection of the relation based on the context.
 * `always_include_linkage_data` - if set to true, the serialized relationship will include the type and id of the related record under a `data` key. Defaults to false if not set.
 * `eager_load_on_include` - if set to false, will not include this relationship in join SQL when requested via an include. You usually want to leave this on, but it will break 'relationships' which are not active record, for example if you want to expose a tree using the `ancestry` gem or similar, or the SQL query becomes too large to handle. Defaults to true if not set.

`to_one` relationships support the additional option:
 * `foreign_key_on` - defaults to `:self`. To indicate that the foreign key is on the related resource specify `:related`.

`to_many` relationships support the additional option:
 * `reflect` - defaults to `true`. To indicate that updates to the relationship are performed on the related resource, if relationship reflection is turned on. See [Configuration] (#configuration)

Examples:

```ruby
class CommentResource < JSONAPI::Resource
  attributes :body
  has_one :post
  has_one :author, class_name: 'Person'
  has_many :tags, acts_as_set: true
end

class ExpenseEntryResource < JSONAPI::Resource
  attributes :cost, :transaction_date

  has_one :currency, class_name: 'Currency', foreign_key: 'currency_code'
  has_one :employee
end

class TagResource < JSONAPI::Resource
  attributes :name
  # The taggable relationship would be serialized with a data node including
  # the `type` and `id` of the taggable instance
  has_one :taggable, polymorphic: true, always_include_linkage_data: true
end
```

```ruby
class BookResource < JSONAPI::Resource

  # Only book_admins may see unapproved comments for a book. Using
  # a lambda to select the correct relation on the model
  has_many :book_comments, relation_name: -> (options = {}) {
    context = options[:context]
    current_user = context ? context[:current_user] : nil

    unless current_user && current_user.book_admin
      :approved_book_comments
    else
      :book_comments
    end
  }
  ...
end
```

The polymorphic relationship will require the resource and controller to exist, although routing to them will cause an error.

```ruby
class TaggableResource < JSONAPI::Resource; end
class TaggablesController < JSONAPI::ResourceController; end
```

## Filters

Filters for locating objects of the resource type are specified in the resource definition. Single filters can be declared using the `filter` method, and multiple filters can be declared with the `filters` method on the resource class.

For example:

```ruby
class ContactResource < JSONAPI::Resource
  attributes :name_first, :name_last, :email, :twitter

  filter :id
  filters :name_first, :name_last
end
```

Then a request could pass in a filter for example `http://example.com/contacts?filter[name_last]=Smith` and the system will find all people where the last name exactly matches Smith.

### Default Filters

A default filter may be defined for a resource using the `default` option on the `filter` method. This default is used unless the request overrides this value.

For example:

```ruby
 class CommentResource < JSONAPI::Resource
  attributes :body, :status
  has_one :post
  has_one :author

  filter :status, default: 'published,pending'
end
```

The default value is used as if it came from the request.

### Applying Filters

You may customize how a filter behaves by supplying a callable to the `:apply` option. This callable will be used to apply that filter. The callable is passed the `records`, which is an `ActiveRecord::Relation`, the `value`, and an `_options` hash. It is expected to return an `ActiveRecord::Relation`.

Note: When a filter is not supplied a `verify` callable to modify the `value` that the `apply` callable receives, `value` defaults to an array of the string values provided to the filter parameter.

This example shows how you can implement different approaches for different filters.

```ruby
# When given the following parameter:'filter[visibility]': 'public'

filter :visibility, apply: ->(records, value, _options) {
  records.where('users.publicly_visible = ?', value[0] == 'public')
}
```

If you omit the `apply` callable the filter will be applied as `records.where(filter => value)`.

Note: It is also possible to override the `self.apply_filter` method, though this approach is now deprecated:

```ruby
def self.apply_filter(records, filter, value, options)
  case filter
    when :last_name, :first_name, :name
      if value.is_a?(Array)
        value.each do |val|
          records = records.where(_model_class.arel_table[filter].matches(val))
        end
        records
      else
        records.where(_model_class.arel_table[filter].matches(value))
      end
    else
      super(records, filter, value)
    end
  end
end
```

### Verifying Filters

Because filters typically come straight from the request, it's prudent to verify their values. To do so, provide a callable to the `verify` option. This callable will be passed the `value` and the `context`. Verify should return the verified value, which may be modified.

```ruby
  filter :ids,
    verify: ->(values, context) {
      verify_keys(values, context)
      values
    },
    apply: ->(records, value, _options) {
      records.where('id IN (?)', value)
    }
```

```ruby
# A more complex example, showing how to filter for any overlap between the
# value array and the possible_ids, using both verify and apply callables.

  filter :possible_ids,
    verify: ->(values, context) {
      values.map {|value| value.to_i}
    },
    apply: ->(records, value, _options) {
      records.where('possible_ids && ARRAY[?]', value)
    }
```

### Finders

Basic finding by filters is supported by resources. This is implemented in the `find` and `find_by_key` finder methods. Currently this is implemented for `ActiveRecord` based resources. The finder methods rely on the `records` method to get an `ActiveRecord::Relation` relation. It is therefore possible to override `records` to affect the three find related methods.

**Note**: The use of caching precludes overriding the finder methods. Please see [Caching Caveats](resource_caching.html#Caching-Caveats) for meore details.

#### Customizing base records for finder methods

If you need to change the base records on which `find` and `find_by_key` operate, you can override the `records` method on the resource class.

For example to allow a user to only retrieve their own posts you can do the following:

```ruby
class PostResource < JSONAPI::Resource
  attributes :title, :body

  def self.records(options = {})
    context = options[:context]
    context[:current_user].posts
  end
end
```

When you create a relationship, a method is created to fetch record(s) for that relationship, using the relation name for the relationship.

```ruby
class PostResource < JSONAPI::Resource
  has_one :author
  has_many :comments

  # def record_for_author
  #   relationship = self.class._relationship(:author)
  #   relation_name = relationship.relation_name(context: @context)
  #   records_for(relation_name)
  # end

  # def records_for_comments
  #   relationship = self.class._relationship(:comments)
  #   relation_name = relationship.relation_name(context: @context)
  #   records_for(relation_name)
  # end
end

```

For example, you may want to raise an error if the user is not authorized to view the related records. See the next section for additional details on raising errors.

```ruby
class BaseResource < JSONAPI::Resource
  def records_for(relation_name)
    context = options[:context]
    records = _model.public_send(relation_name)

    unless context[:current_user].can_view?(records)
      raise NotAuthorizedError
    end

    records
  end
end
```

#### Raising Errors

Inside the finder methods (like `records_for`) or inside of resource callbacks (like `before_save`) you can `raise` an error to halt processing. JSONAPI::Resources has some built in errors that will return appropriate error codes. By default any other error that you raise will return a `500` status code for a general internal server error.

To return useful error codes that represent application errors you should set the `exception_class_whitelist` config variable, and then you should use the Rails `rescue_from` macro to render a status code.

For example, this config setting allows the `NotAuthorizedError` to bubble up out of JSONAPI::Resources and into your application.

```ruby
# config/initializer/jsonapi-resources.rb
JSONAPI.configure do |config|
  config.exception_class_whitelist = [NotAuthorizedError]
end
```

Handling the error and rendering the appropriate code is now the responsibility of the application and could be handled like this:

```ruby
class ApiController < ApplicationController
  rescue_from NotAuthorizedError, with: :reject_forbidden_request
  def reject_forbidden_request
    render json: {error: 'Forbidden'}, :status => 403
  end
end
```


#### Applying Filters

The `apply_filter` method is called to apply each filter to the `Arel` relation. You may override this method to gain control over how the filters are applied to the `Arel` relation.

This example shows how you can implement different approaches for different filters.

```ruby
def self.apply_filter(records, filter, value, options)
  case filter
    when :visibility
      records.where('users.publicly_visible = ?', value == :public)
    when :last_name, :first_name, :name
      if value.is_a?(Array)
        value.each do |val|
          records = records.where(_model_class.arel_table[filter].matches(val))
        end
        records
      else
        records.where(_model_class.arel_table[filter].matches(value))
      end
    else
      super(records, filter, value)
    end
  end
end
```

#### Applying Sorting

You can override the `apply_sort` method to gain control over how the sorting is done. This may be useful in case you'd like to base the sorting on variables in your context.

Example:

```ruby
def self.apply_sort(records, order_options, context = {})
  if order_options.has?(:trending)
    records = records.order_by_trending_scope
    order_options - [:trending]
  end

  super(records, order_options, context)
end
```


#### Override Finder Methods

Finally, if you have more complex requirements for finding, you can override the `find` and `find_by_key` methods on the resource class.

Here's an example that defers the `find` operation to a `current_user` set on the `context` option:

```ruby
class AuthorResource < JSONAPI::Resource
  attribute :name
  model_name 'Person'
  has_many :posts

  filter :name

  def self.find(filters, options = {})
    context = options[:context]
    authors = context[:current_user].find_authors(filters)

    return authors.map do |author|
      self.new(author, context)
    end
  end
end
```

## Custom Sorting

You can override the `apply_sort` method to gain control over how the sorting is done. This may be useful in case you'd like to base the sorting on variables in your context.

Example:

```ruby
def self.apply_sort(records, order_options, context = {})
  if order_options.has_key?('trending')
    records = records.order_by_trending_scope
    order_options.delete('trending')
  end

  super(records, order_options, context)
end
```

You can also sort record sets in-memory instead of via Active Record.

Example:

```ruby
def self.apply_sort(records, order_options, context = {})
  if order_options.has_key?('trending')
    records = records.to_a.sort_by { |r| r.considered_trending?(context[:some_query_arg]) }
    records.reverse! if order_options['trending'] == :desc

    order_options.delete('trending')
  end

  super(records, order_options, context)
end
```


## Pagination

Pagination is performed using a `paginator`, which is a class responsible for parsing the `page` request parameters and applying the pagination logic to the results.

### Paginators

`JSONAPI::Resource` supports several pagination methods by default, and it allows you to implement a custom system if the defaults do not meet your needs.

#### Paged Paginator

The `paged` `paginator` returns results based on pages of a fixed size. Valid `page` parameters are `number` and `size`. If `number` is omitted the first page is returned. If `size` is omitted the `default_page_size` from the configuration settings is used.

```
GET /articles?page%5Bnumber%5D=10&page%5Bsize%5D=10 HTTP/1.1
Accept: application/vnd.api+json
```

#### Offset Paginator

The `offset` `paginator` returns results based on an offset from the beginning of the resultset. Valid `page` parameters are `offset` and `limit`. If `offset` is omitted a value of 0 will be used. If `limit` is omitted the `default_page_size` from the configuration settings is used.

```
GET /articles?page%5Blimit%5D=10&page%5Boffset%5D=10 HTTP/1.1
Accept: application/vnd.api+json
```

#### Custom Paginators

Custom `paginators` can be used. These should derive from `Paginator`. The `apply` method takes a `relation` and `order_options` and is expected to return a `relation`. The `initialize` method receives the parameters from the `page` request parameters. It is up to the paginator author to parse and validate these parameters.

For example, here is a very simple single record at a time paginator:

```ruby
class SingleRecordPaginator < JSONAPI::Paginator
  def initialize(params)
    # param parsing and validation here
    @page = params.to_i
  end

  def apply(relation, order_options)
    relation.offset(@page).limit(1)
  end
end
```

### Paginator Configuration

The default paginator, which will be used for all resources, is set using `JSONAPI.configure`. For example, in your `config/initializers/jsonapi_resources.rb`:

```ruby
JSONAPI.configure do |config|
  # built in paginators are :none, :offset, :paged
  config.default_paginator = :offset

  config.default_page_size = 10
  config.maximum_page_size = 20
end
```

If no `default_paginator` is configured, pagination will be disabled by default.

Paginators can also be set at the resource-level, which will override the default setting. This is done using the `paginator` method:

```ruby
class BookResource < JSONAPI::Resource
  attribute :title
  attribute :isbn

  paginator :offset
end
```

To disable pagination in a resource, specify `:none` for `paginator`.

## Included relationships (side-loading resources)

JR supports [request include params](http://jsonapi.org/format/#fetching-includes) out of the box, for side loading related resources.

Here's an example from the spec:

```
GET /articles/1?include=comments HTTP/1.1
Accept: application/vnd.api+json
```

Will get you the following payload by default:

```
{
  "data": {
    "type": "articles",
    "id": "1",
    "attributes": {
      "title": "JSON API paints my bikeshed!"
    },
    "links": {
      "self": "http://example.com/articles/1"
    },
    "relationships": {
      "comments": {
        "links": {
          "self": "http://example.com/articles/1/relationships/comments",
          "related": "http://example.com/articles/1/comments"
        },
        "data": [
          { "type": "comments", "id": "5" },
          { "type": "comments", "id": "12" }
        ]
      }
    }
  },
  "included": [{
    "type": "comments",
    "id": "5",
    "attributes": {
      "body": "First!"
    },
    "links": {
      "self": "http://example.com/comments/5"
    }
  }, {
    "type": "comments",
    "id": "12",
    "attributes": {
      "body": "I like XML better"
    },
    "links": {
      "self": "http://example.com/comments/12"
    }
  }]
}
```

Note: When passing `include` and `fields` params together, relationships not included in the `fields` parameter will not be serialized. This will have the side effect of not serializing the included resources. To ensure the related resources are properly side loaded specify them in the `fields`, like `fields[posts]=comments,title&include=comments`.

## Resource Meta

Meta information can be included for each resource using the meta method in the resource declaration. For example:

```ruby
class BookResource < JSONAPI::Resource
  attribute :title
  attribute :isbn

  def meta(options)
    {
      copyright: 'API Copyright 2015 - XYZ Corp.',
      computed_copyright: options[:serialization_options][:copyright],
      last_updated_at: _model.updated_at
    }
   end
end

```

The `meta` method will be called for each resource instance. Override the `meta` method on a resource class to control the meta information for the resource. If a non empty hash is returned from `meta` this will be serialized. The `meta` method is called with an `options` hash. The `options` hash will contain the following:

 * `:serializer` -> the serializer instance
 * `:serialization_options` -> the contents of the `serialization_options` method on the controller.

## Custom Links

Custom links can be included for each resource by overriding the `custom_links` method. If a non empty hash is returned from `custom_links`, it will be merged with the default links hash containing the resource's `self` link. The `custom_links` method is called with the same `options` hash used by for [resource meta information](#resource-meta). The `options` hash contains the following:

 * `:serializer` -> the serializer instance
 * `:serialization_options` -> the contents of the `serialization_options` method on the controller.

For example:

```ruby
class CityCouncilMeeting < JSONAPI::Resource
  attribute :title, :location, :approved

  def custom_links(options)
    { minutes: options[:serializer].link_builder.self_link(self) + "/minutes" }
  end
end
```

This will create a custom link with the key `minutes`, which will be merged with the default `self` link, like so:

```json
{
  "data": [
    {
      "id": "1",
      "type": "cityCouncilMeetings",
      "links": {
        "self": "http://city.gov/api/city-council-meetings/1",
        "minutes": "http://city.gov/api/city-council-meetings/1/minutes"
      },
      "attributes": {...}
    },
    //...
  ]
}
```

Of course, the `custom_links` method can include logic to include links only when relevant:

```ruby
class CityCouncilMeeting < JSONAPI::Resource
  attribute :title, :location, :approved

  delegate :approved?, to: :model

  def custom_links(options)
    extra_links = {}
    if approved?
      extra_links[:minutes] = options[:serializer].link_builder.self_link(self) + "/minutes"
    end
    extra_links
  end
end
```

It's also possibly to suppress the default `self` link by returning a hash with `{self: nil}`:

```ruby
class Selfless < JSONAPI::Resource
  def custom_links(options)
    {self: nil}
  end
end
```

## Callbacks

`ActiveSupport::Callbacks` is used to provide callback functionality, so the behavior is very similar to what you may be used to from `ActiveRecord`.

For example, you might use a callback to perform authorization on your resource before an action.

```ruby
class BaseResource < JSONAPI::Resource
  before_create :authorize_create

  def authorize_create
    # ...
  end
end
```

The types of supported callbacks are:
- `before`
- `after`
- `around`

### `JSONAPI::Resource` Callbacks

Callbacks can be defined for the following `JSONAPI::Resource` events:

- `:create`
- `:update`
- `:remove`
- `:save`
- `:create_to_many_link`
- `:replace_to_many_links`
- `:create_to_one_link`
- `:replace_to_one_link`
- `:remove_to_many_link`
- `:remove_to_one_link`
- `:replace_fields`

#### Relationship Reflection

By default updates to relationships only invoke callbacks on the primary Resource. By setting the `use_relationship_reflection` [Configuration] (#configuration) option updates to `has_many` relationships will occur on the related resource, triggering callbacks on both resources.

### `JSONAPI::Processor` Callbacks

Callbacks can also be defined for `JSONAPI::Processor` events:
- `:operation`: Any individual operation.
- `:find`: A `find` operation is being processed.
- `:show`: A `show` operation is being processed.
- `:show_relationship`: A `show_relationship` operation is being processed.
- `:show_related_resource`: A `show_related_resource` operation is being processed.
- `:show_related_resources`: A `show_related_resources` operation is being processed.
- `:create_resource`: A `create_resource` operation is being processed.
- `:remove_resource`: A `remove_resource` operation is being processed.
- `:replace_fields`: A `replace_fields` operation is being processed.
- `:replace_to_one_relationship`: A `replace_to_one_relationship` operation is being processed.
- `:create_to_many_relationship`: A `create_to_many_relationship` operation is being processed.
- `:replace_to_many_relationship`: A `replace_to_many_relationship` operation is being processed.
- `:remove_to_many_relationship`: A `remove_to_many_relationship` operation is being processed.
- `:remove_to_one_relationship`: A `remove_to_one_relationship` operation is being processed.

See [Operation Processors](operation_processors.html) for details on using OperationProcessors.

### `JSONAPI::OperationsProcessor` Callbacks (a removed feature)

**Note:** The `JSONAPI::OperationsProcessor` has been removed and replaced with the `JSONAPI::OperationDispatcher` and `Processor` classes per resource. The callbacks have been renamed and moved to the `Processor`s, with the exception of the `operations` callback which is now on the controller.
