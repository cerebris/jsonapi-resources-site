title: Contributing to the Project
type: guide
order: 100
version: 0.9
---

1. Fork it ( http://github.com/cerebris/jsonapi-resources/fork )
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create a new Pull Request

### Running Tests

To run the tests for this project:

- `rake test` or `bundle exec rake test`

To run a single test:

- `bundle exec ruby -I test test/controllers/controller_test.rb -n test_type_formatting`
