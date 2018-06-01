title: Controllers
type: guide
order: 30
version: 0.10
---

There are two ways to implement a controller for your resources. Either derive from `ResourceController` or import the `ActsAsResourceController` module.

## ResourceController

`JSONAPI::Resources` provides a class, `ResourceController`, that can be used as the base class for your controllers. `ResourceController` supports `index`, `show`, `create`, `update`, and `destroy` methods. Just deriving your controller from `ResourceController` will give you a fully functional controller.

For example:

```ruby
class PeopleController < JSONAPI::ResourceController

end
```

Of course you are free to extend this as needed and override action handlers or other methods.

A jsonapi-controller generator is avaliable:

```
rails generate jsonapi:controller contact
```

### ResourceControllerMetal

`JSONAPI::Resources` also provides an alternative class to `ResourceController` called `ResourceControllerMetal`. In order to provide a lighter weight controller option this strips the controller down to just the classes needed to work with `JSONAPI::Resources`.

For example:

```ruby
class PeopleController < JSONAPI::ResourceControllerMetal

end
```

Note: This may not provide all of the expected controller capabilities if you are using additional gems such as DoorKeeper.

### Serialization Options

Additional options can be passed to the serializer using the `serialization_options` method.

For example:

```ruby
class ApplicationController < JSONAPI::ResourceController
  def serialization_options
    {copyright: 'Copyright 2015'}
  end
end
```

These `serialization_options` are passed to the `meta` method used to generate resource `meta` values.

## ActsAsResourceController

`JSONAPI::Resources` also provides a module, `JSONAPI::ActsAsResourceController`. You can include this module to mix in all the features of `ResourceController` into your existing controller class.

For example:

```ruby
class PostsController < ActionController::Base
  include JSONAPI::ActsAsResourceController
end
```

#### Namespaces

JSONAPI::Resources supports namespacing of controllers and resources. With namespacing, you can version your API.

If you namespace your controller, it will require a namespaced resource.

In the following example, we have a `resource` that isn't namespaced, and one that has now been namespaced. There are slight differences between the two resources, as might be seen in a new version of an API:

```ruby
class PostResource < JSONAPI::Resource
  attribute :title
  attribute :body
  attribute :subject

  has_one :author, class_name: 'Person'
  has_one :section
  has_many :tags, acts_as_set: true
  has_many :comments, acts_as_set: false
  def subject
    @model.title
  end

  filters :title, :author, :tags, :comments
  filter :id
end

...

module Api
  module V1
    class PostResource < JSONAPI::Resource
      # V1 replaces the non-namespaced resource
      # V1 no longer supports tags and now calls author 'writer'
      attribute :title
      attribute :body
      attribute :subject

      has_one :writer, foreign_key: 'author_id'
      has_one :section
      has_many :comments, acts_as_set: false

      def subject
        @model.title
      end

      filters :writer
    end

    class WriterResource < JSONAPI::Resource
      attributes :name, :email
      model_name 'Person'
      has_many :posts

      filter :name
    end
  end
end
```

The following controllers are used:

```ruby
class PostsController < JSONAPI::ResourceController
end

module Api
  module V1
    class PostsController < JSONAPI::ResourceController
    end
  end
end
```

You will also need to namespace your routes:

```ruby
Rails.application.routes.draw do

  jsonapi_resources :posts

  namespace :api do
    namespace :v1 do
      jsonapi_resources :posts
    end
  end
end
```

When a namespaced `resource` is used, any related `resources` must also be in the same namespace.

#### Error codes

Error codes are provided for each error object returned, based on the error. These errors are:

```ruby
module JSONAPI
  VALIDATION_ERROR = '100'
  INVALID_RESOURCE = '101'
  FILTER_NOT_ALLOWED = '102'
  INVALID_FIELD_VALUE = '103'
  INVALID_FIELD = '104'
  PARAM_NOT_ALLOWED = '105'
  PARAM_MISSING = '106'
  INVALID_FILTER_VALUE = '107'
  COUNT_MISMATCH = '108'
  KEY_ORDER_MISMATCH = '109'
  KEY_NOT_INCLUDED_IN_URL = '110'
  INVALID_INCLUDE = '112'
  RELATION_EXISTS = '113'
  INVALID_SORT_CRITERIA = '114'
  INVALID_LINKS_OBJECT = '115'
  TYPE_MISMATCH = '116'
  INVALID_PAGE_OBJECT = '117'
  INVALID_PAGE_VALUE = '118'
  INVALID_FIELD_FORMAT = '119'
  INVALID_FILTERS_SYNTAX = '120'
  SAVE_FAILED = '121'
  FORBIDDEN = '403'
  RECORD_NOT_FOUND = '404'
  NOT_ACCEPTABLE = '406'
  UNSUPPORTED_MEDIA_TYPE = '415'
  LOCKED = '423'
end
```

These codes can be customized in your app by creating an initializer to override any or all of the codes.

In addition, textual error codes can be returned by setting the configuration option `use_text_errors = true`. For example:

```ruby
JSONAPI.configure do |config|
  config.use_text_errors = true
end
```


#### Handling Exceptions

By default, all exceptions raised downstream from a resource controller will be caught, logged, and a `500 Internal Server Error` will be rendered. Exceptions can be whitelisted in the config to pass through the handler and be caught manually, or you can pass a callback from a resource controller to insert logic into the rescue block without interrupting the control flow. This can be particularly useful for additional logging or monitoring without the added work of rendering responses.

Pass a block, refer to controller class methods, or both. Note that methods must be defined as class methods on a controller and accept one parameter, which is passed the exception object that was rescued.

```ruby
  class ApplicationController < JSONAPI::ResourceController

    on_server_error :first_callback

    #or

    # on_server_error do |error|
      #do things
    #end

    def self.first_callback(error)
      #env["airbrake.error_id"] = notify_airbrake(error)
    end
  end

```

#### Action Callbacks

## verify_content_type_header

By default, when controllers extend functionalities from `jsonapi-resources`, the `ActsAsResourceController#verify_content_type_header` method will be triggered before `create`, `update`, `create_relationship` and `update_relationship` actions. This method is responsible for checking if client's request corresponds to the correct media type required by [JSON API](http://jsonapi.org/format/#content-negotiation-clients): `application/vnd.api+json`.

In case you need to check the media type for custom actions, just make sure to call the method in your controller's `before_action`:

```ruby
class UsersController < JSONAPI::ResourceController
  before_action :verify_content_type_header, only: [:auth]

  def auth
    # some crazy auth code goes here
  end
end
```
