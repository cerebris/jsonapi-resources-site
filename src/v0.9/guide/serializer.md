title: Serializer
type: guide
order: 50
version: 0.9
---

The `ResourceSerializer` can be used to serialize a resource into JSON API compliant JSON. `ResourceSerializer` must be initialized with the primary resource type it will be serializing. `ResourceSerializer` has a `serialize_to_hash` method that takes a resource instance or array of resource instances to serialize. For example:

```ruby
post = Post.find(1)
JSONAPI::ResourceSerializer.new(PostResource).serialize_to_hash(PostResource.new(post, nil))
```

Note: If your resource needs to access to state from a context hash, make sure to pass the context hash as the second argument of the resource class new method. For example:

```ruby
post = Post.find(1)
context = { current_user: current_user }
JSONAPI::ResourceSerializer.new(PostResource).serialize_to_hash(PostResource.new(post, context))
```

This returns results like this:

```json
{
  "data": {
    "type": "posts",
    "id": "1",
    "links": {
      "self": "http://example.com/posts/1"
    },
    "attributes": {
      "title": "New post",
      "body": "A body!!!",
      "subject": "New post"
    },
    "relationships": {
      "section": {
        "links": {
          "self": "http://example.com/posts/1/relationships/section",
          "related": "http://example.com/posts/1/section"
        },
        "data": null
      },
      "author": {
        "links": {
          "self": "http://example.com/posts/1/relationships/author",
          "related": "http://example.com/posts/1/author"
        },
        "data": {
          "type": "people",
          "id": "1"
        }
      },
      "tags": {
        "links": {
          "self": "http://example.com/posts/1/relationships/tags",
          "related": "http://example.com/posts/1/tags"
        }
      },
      "comments": {
        "links": {
          "self": "http://example.com/posts/1/relationships/comments",
          "related": "http://example.com/posts/1/comments"
        }
      }
    }
  }
}
```

## Options

The `ResourceSerializer` can be initialized with some optional parameters:

### `include`

An array of resources. Nested resources can be specified with dot notation.

  *Purpose*: determines which objects will be side loaded with the source objects in an `included` section

  *Example*: `include: ['comments','author','comments.tags','author.posts']`

### `fields`

A hash of resource types and arrays of fields for each resource type.

  *Purpose*: determines which fields are serialized for a resource type. This encompasses both attributes and relationship ids in the links section for a resource. Fields are global for a resource type.

  *Example*: `fields: { people: [:email, :comments], posts: [:title, :author], comments: [:body, :post]}`

```ruby
post = Post.find(1)
include_resources = ['comments','author','comments.tags','author.posts']

JSONAPI::ResourceSerializer.new(PostResource, include: include_resources,
  fields: {
    people: [:email, :comments],
    posts: [:title, :author],
    tags: [:name],
    comments: [:body, :post]
  }
).serialize_to_hash(PostResource.new(post, nil))
```
