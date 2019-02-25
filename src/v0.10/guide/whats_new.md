title: What's New
type: guide
order: 2
version: 0.10
---

Version 0.10 brings a major refactoring to much of the library internals. The
primary changes are around resource retrieval.

**Note: These docs are being updated gradually for this new release. This will
be an ongoing activity.**

## Upgrade guide

A new [Upgrade Guide](upgrade_guide.html) has been added to these docs.

## Major changes

### Resource finders

In this release the concerns of the `Resource` class that interact with
`ActiveRelation` have been moved into a module named
`ActiveRelationResourceFinder`. Resource finders provide a common interface for
the processor to efficiently build a tree of resources for the request, taking
into account caching, includes, filters, sorting and pagination.

Note that `ActiveRelationResourceFinder` is the default and currently only
resource finder for the system, but it's expected that additional resource
finders will be written to handle other data sources.

### Processors

The `Processor` class has changed significantly in how it builds the request's
result. For `show` and `index` requests the Processor works with the resource
finder to first find the identifiers and optional cache fields for the primary
and included resources, represented as `ResourceFragment`s. If a resource
enables caching the cached resources are pulled for those resources without
changes. Finally the full `ResourceSet` is populated from the data source for any
cache misses. The `ActiveRelationResourceFinder` uses pluck for the first stage
to get the identifiers and cache fields. This avoids the overhead of model
instantiation for cache hits.

### New helper classes

Helper classes for the processor and resource finders have been introduced.
These are:

* `ResourceIdentity`
* `ResourceFragment`
* `ResourceIdTree`
* `ResourceSet`

Paths for includes, filters and sorts are now handled by the `Path` and
`PathSegment` classes.

## Breaking changes

### Sort and filter `apply` callables

Resources that use custom `apply` callables for filters and sorts may need
changes. Previously assumptions on table aliases had to be made. This could lead
to filters being applied on the wrong tables, or creating invalid SQL if
ActiveRecord constructed the query differently than expected. The system now
tries to create as few joins as possible to perform the request, and tracks the
aliases used. This is passed into the callables as an option. See
[Applying-Filters](resources.html#Applying-Filters) for more details.

### `find_records` removed

In rare cases you might have overridden the `Resource.find_records` method. This
method is no longer called. It's recommended that you use the `apply` callable
on filters and sorts, the `apply_join` callable on relationships, and you can
continue overriding the `self.records` method.

### `records_for` removed

The `records_for` method has been removed. Related records should now use the
`apply_join` callable on relationships.

### `apply_includes` removed

Resources that override the `Resource.apply_includes` method should be aware
that this is no longer going to be called. This is due to the internal changes
where related resources are no longer accessed through the model's
auto-generated association methods for finding resources.
