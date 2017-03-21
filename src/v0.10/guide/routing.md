title: Routing
type: guide
order: 60
version: 0.10
---

## Helper Methods

JR has a couple of helper methods available to assist you with setting up routes.

### `jsonapi_resources`

Like `resources` in `ActionDispatch`, `jsonapi_resources` provides resourceful routes mapping between HTTP verbs and URLs and controller actions. This will also setup mappings for relationship URLs for a resource's relationships. For example:

```ruby
Rails.application.routes.draw do
  jsonapi_resources :contacts
  jsonapi_resources :phone_numbers
end
```

gives the following routes

```
                     Prefix Verb      URI Pattern                                               Controller#Action
contact_relationships_phone_numbers GET       /contacts/:contact_id/relationships/phone-numbers(.:format)       contacts#show_relationship {:relationship=>"phone_numbers"}
                            POST      /contacts/:contact_id/relationships/phone-numbers(.:format)       contacts#create_relationship {:relationship=>"phone_numbers"}
                            DELETE    /contacts/:contact_id/relationships/phone-numbers/:keys(.:format) contacts#destroy_relationship {:relationship=>"phone_numbers"}
      contact_phone_numbers GET       /contacts/:contact_id/phone-numbers(.:format)             phone_numbers#get_related_resources {:relationship=>"phone_numbers", :source=>"contacts"}
                   contacts GET       /contacts(.:format)                                       contacts#index
                            POST      /contacts(.:format)                                       contacts#create
                    contact GET       /contacts/:id(.:format)                                   contacts#show
                            PATCH     /contacts/:id(.:format)                                   contacts#update
                            PUT       /contacts/:id(.:format)                                   contacts#update
                            DELETE    /contacts/:id(.:format)                                   contacts#destroy
 phone_number_relationships_contact GET       /phone-numbers/:phone_number_id/relationships/contact(.:format)   phone_numbers#show_relationship {:relationship=>"contact"}
                            PUT|PATCH /phone-numbers/:phone_number_id/relationships/contact(.:format)   phone_numbers#update_relationship {:relationship=>"contact"}
                            DELETE    /phone-numbers/:phone_number_id/relationships/contact(.:format)   phone_numbers#destroy_relationship {:relationship=>"contact"}
       phone_number_contact GET       /phone-numbers/:phone_number_id/contact(.:format)         contacts#get_related_resource {:relationship=>"contact", :source=>"phone_numbers"}
              phone_numbers GET       /phone-numbers(.:format)                                  phone_numbers#index
                            POST      /phone-numbers(.:format)                                  phone_numbers#create
               phone_number GET       /phone-numbers/:id(.:format)                              phone_numbers#show
                            PATCH     /phone-numbers/:id(.:format)                              phone_numbers#update
                            PUT       /phone-numbers/:id(.:format)                              phone_numbers#update
                            DELETE    /phone-numbers/:id(.:format)                              phone_numbers#destroy
```

Like `resources`, you may specify `only:` or `except:`, like `jsonapi_resources :contacts, only: [:create]`.

### `jsonapi_resource`

Like `jsonapi_resources`, but for resources you lookup without an id.

## Nested Routes

By default nested routes are created for getting related resources and manipulating relationships. You can control the nested routes by passing a block into `jsonapi_resources` or `jsonapi_resource`. An empty block will not create any nested routes. For example:

```ruby
Rails.application.routes.draw do
  jsonapi_resources :contacts do
  end
end
```

gives routes that are only related to the primary resource, and none for its relationships:

```
      Prefix Verb   URI Pattern                  Controller#Action
    contacts GET    /contacts(.:format)          contacts#index
             POST   /contacts(.:format)          contacts#create
     contact GET    /contacts/:id(.:format)      contacts#show
             PATCH  /contacts/:id(.:format)      contacts#update
             PUT    /contacts/:id(.:format)      contacts#update
             DELETE /contacts/:id(.:format)      contacts#destroy
```

To manually add in the nested routes you can use the `jsonapi_links`, `jsonapi_related_resources` and `jsonapi_related_resource` inside the block. Or, you can add the default set of nested routes using the `jsonapi_relationships` method. For example:

```ruby
Rails.application.routes.draw do
  jsonapi_resources :contacts do
    jsonapi_relationships
  end
end
```

### `jsonapi_links`

You can add relationship routes in with `jsonapi_links`, for example:

```ruby
Rails.application.routes.draw do
  jsonapi_resources :contacts do
    jsonapi_links :phone_numbers
  end
end
```

Gives the following routes:

```
contact_relationships_phone_numbers GET    /contacts/:contact_id/relationships/phone-numbers(.:format)       contacts#show_relationship {:relationship=>"phone_numbers"}
                            POST   /contacts/:contact_id/relationships/phone-numbers(.:format)       contacts#create_relationship {:relationship=>"phone_numbers"}
                            DELETE /contacts/:contact_id/relationships/phone-numbers/:keys(.:format) contacts#destroy_relationship {:relationship=>"phone_numbers"}
                   contacts GET    /contacts(.:format)                                       contacts#index
                            POST   /contacts(.:format)                                       contacts#create
                    contact GET    /contacts/:id(.:format)                                   contacts#show
                            PATCH  /contacts/:id(.:format)                                   contacts#update
                            PUT    /contacts/:id(.:format)                                   contacts#update
                            DELETE /contacts/:id(.:format)                                   contacts#destroy

```

The new routes allow you to show, create and destroy the relationships between resources.

### `jsonapi_related_resources`

Creates a nested route to GET the related has_many resources. For example:

```ruby
Rails.application.routes.draw do
  jsonapi_resources :contacts do
    jsonapi_related_resources :phone_numbers
  end
end

```

gives the following routes:

```
               Prefix Verb   URI Pattern                                   Controller#Action
contact_phone_numbers GET    /contacts/:contact_id/phone-numbers(.:format) phone_numbers#get_related_resources {:relationship=>"phone_numbers", :source=>"contacts"}
             contacts GET    /contacts(.:format)                           contacts#index
                      POST   /contacts(.:format)                           contacts#create
              contact GET    /contacts/:id(.:format)                       contacts#show
                      PATCH  /contacts/:id(.:format)                       contacts#update
                      PUT    /contacts/:id(.:format)                       contacts#update
                      DELETE /contacts/:id(.:format)                       contacts#destroy

```

A single additional route was created to allow you GET the phone numbers through the contact.

### `jsonapi_related_resource`

Like `jsonapi_related_resources`, but for has_one related resources.

```ruby
Rails.application.routes.draw do
  jsonapi_resources :phone_numbers do
    jsonapi_related_resource :contact
  end
end
```

gives the following routes:

```
              Prefix Verb   URI Pattern                                       Controller#Action
phone_number_contact GET    /phone-numbers/:phone_number_id/contact(.:format) contacts#get_related_resource {:relationship=>"contact", :source=>"phone_numbers"}
       phone_numbers GET    /phone-numbers(.:format)                          phone_numbers#index
                     POST   /phone-numbers(.:format)                          phone_numbers#create
        phone_number GET    /phone-numbers/:id(.:format)                      phone_numbers#show
                     PATCH  /phone-numbers/:id(.:format)                      phone_numbers#update
                     PUT    /phone-numbers/:id(.:format)                      phone_numbers#update
                     DELETE /phone-numbers/:id(.:format)                      phone_numbers#destroy

```
