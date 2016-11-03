title: Operation Processors
type: guide
order: 40
version: 0.9
---

Operation Processors are called to perform the operation(s) that make up a request. The controller (through the `OperationDispatcher`), creates an `OperatorProcessor` to handle each operation. The processor is created based on the resource name, including the namespace. If a processor does not exist for a resource (namespace matters) the default operation processor is used instead. The default processor can be changed by a configuration setting.

## Custom OperatorProcessors

Defining a custom `Processor` allows for custom callback handling of each operation type for each resource type. For example:

```ruby
class Api::V4::BookProcessor < JSONAPI::Processor
  after_find do
    unless @result.is_a?(JSONAPI::ErrorsOperationResult)
      @result.meta[:total_records_found] = @result.record_count
    end
  end
end
```

This simple example uses a callback to update the result's meta property with the total count of records (a redundant feature only for example purposes), if there wasn't an error in the operation.  It is also possible to override the `find` method as well if a different behavior is needed, for example:

```ruby
class Api::V4::BookProcessor < JSONAPI::Processor
  def find
    filters = params[:filters]
    include_directives = params[:include_directives]
    sort_criteria = params.fetch(:sort_criteria, [])
    paginator = params[:paginator]

    verified_filters = resource_klass.verify_filters(filters, context)
    resource_records = resource_klass.find(verified_filters,
                                           context: context,
                                           include_directives: include_directives,
                                           sort_criteria: sort_criteria,
                                           paginator: paginator)

    page_options = {}
    # Overriding the default record count logic to always include it in the meta
    #if (JSONAPI.configuration.top_level_meta_include_record_count ||
    #  (paginator && paginator.class.requires_record_count))
      page_options[:record_count] = resource_klass.find_count(verified_filters,
                                                              context: context,
                                                              include_directives: include_directives)
    #end
end
```

**Note:** The authors of this gem expect the most common uses cases to be handled using the callbacks. It is likely that the internal functionality of the operation processing methods will change, at least for several revisions. Effort will be made to call this out in release notes. You have been warned.
