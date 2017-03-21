title: What's New
type: guide
order: 2
version: 0.9
---

## Features
* [Caching support for ActiveRecord based resources](resource_caching.html)
* Ids are now fetched through a resource method which delegates to the model
* Requests are now handled internally in one operation.
* Exception backtraces and now configurable instead of defaulting to `!production`.
* Configuration option to whitelist all exceptions.
* Loosens Accept header check to allow header to start with :api_json

## Breaking Changes
* Removed deprecated `updateable` and `createable` methods (note misspellings). Use `updatable` and `creatable` instead.
* Drops support for Rails 4.1
* Requires Ruby 2.1
* Derived resources now use the model name of their base resource, if it is not abstract.
* When using client generated ids, e.g. for GUIDs, the `id` field must be added to the `creatable_fields` method.

And finally there were numerous bug fixes. Full details are in the [Release Notes](https://github.com/cerebris/jsonapi-resources/releases/tag/v0.9.0).