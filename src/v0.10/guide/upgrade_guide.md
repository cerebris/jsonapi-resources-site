title: Upgrade Guide
type: guide
order: 4
version: 0.10
---

## Upgrading Gemfile

```ruby
  gem 'jsonapi-resources', '~>0.10.0.beta8'

  # Use alpha level code at your own risk
  # gem 'jsonapi-resources', git: 'https://github.com/cerebris/jsonapi-resources.git', branch: 'master'
```

And then execute:

```bash
bundle
```

## Check for known upgrade issues

Run the `jsonapi:resources:check_upgrade` rake task:

```bash
rake jsonapi:resources:check_upgrade
```

This will look for overrides in your existing resources that will no longer
be called by JR.

### Fix the resources detected by the check

#### `find_records`

The `find_records` method has been removed and the functionality has been moved
to other methods. It's recommended that you use the `apply` callable on filters
and sorts, and the `apply_join` callable on relationships.

#### `records`

While the `records` method has not been changed some of the overrides that
were needed for custom app logic may need to be reconsidered. One area in
particular is joining or including to_many relationships. If this can't be
avoided it is recommended that a `distinct` call be added to the relation.

The new configuration option `warn_on_performance_issues` is designed to catch
this state.

#### `records_for`

The `records_for` method has been removed. Related records should now use the
`apply_join` callable on relationships.

#### `apply_includes`

The `apply_includes` has been removed. Because related resources are no longer
loaded using the rails auto-generated association methods the need for
`includes` to prevent N+1 query execution has been removed. If you need
`includes` for a resource, say for a calculated attribute using a value on a
related model, you can add the call to `includes` in an override of
`self.records`.

### Processors

The `Processor` class has changed significantly in how it builds the request's
result. For `show` and `index` requests the Processor works with the resource
finder to first find the identifiers and optional cache fields for the primary
and included resources, represented as a `ResourceFragment`. If a resource
enables caching the cached resources are pulled for those resources without
changes. Finally the full resource set is populated from the data source for any
cache misses. The `ActiveRelationResourceFinder` uses pluck for the first stage
to get the identifiers and cache fields. This avoids the overhead of model
instantiation for cache hits.

If you have created custom processors or overridden features you will need to
update these.

### Additional steps

*** this will be updated as needed ***

Please report issues you encounter upgrading either at
[Gitter](https://gitter.im/cerebris/jsonapi-resources) or
[Github](https://github.com/cerebris/jsonapi-resources/issues)
