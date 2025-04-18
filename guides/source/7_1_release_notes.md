**DO NOT READ THIS FILE ON GITHUB, GUIDES ARE PUBLISHED ON https://guides.rubyonrails.org.**

Ruby on Rails 7.1 Release Notes
===============================

Highlights in Rails 7.1:

--------------------------------------------------------------------------------

Upgrading to Rails 7.1
----------------------

If you're upgrading an existing application, it's a great idea to have good test
coverage before going in. You should also first upgrade to Rails 7.0 in case you
haven't and make sure your application still runs as expected before attempting
an update to Rails 7.1. A list of things to watch out for when upgrading is
available in the
[Upgrading Ruby on Rails](upgrading_ruby_on_rails.html#upgrading-from-rails-7-0-to-rails-7-1)
guide.

Major Features
--------------

### Generate Dockerfiles for new Rails applications

[Default Docker support](https://github.com/rails/rails/pull/46762) to new rails applications.
When generating a new application, Rails will now include Docker-related files in the application.

These files serve as a foundational setup for deploying your Rails application in a
production environment using Docker. It's important to note that these files are not
meant for development purposes.

Here's a quick example of how to build and run your Rails app using these Docker files:

```bash
docker build -t app .
docker volume create app-storage
docker run --rm -it -v app-storage:/rails/storage -p 3000:3000 --env RAILS_MASTER_KEY=<your-config-master-key> app
```

You can also start a console or runner from this Docker image:

```bash
docker run --rm -it -v app-storage:/rails/storage --env RAILS_MASTER_KEY=<your-config-master-key> app console
```

For those looking to create a multi-platform image (e.g., Apple Silicon for AMD or Intel deployment),
and push it to Docker Hub, follow these steps:

```bash
docker login -u <your-user>
docker buildx create --use
docker buildx build --push --platform=linux/amd64,linux/arm64 -t <your-user/image-name> .
```

This enhancement simplifies the deployment process, providing a convenient starting point for
getting your Rails application up and running in a production environment.

### Add `ActiveRecord::Base.normalizes`

Normalizations can be declared for attribute values. The normalization
takes place when the attribute is assigned or updated, and will be persisted to the database.
Normalization is also applied to corresponding keyword arguments in finder methods,
allowing records to be queried using unnormalized values.

For example:

```ruby
class User < ActiveRecord::Base
  normalizes :email, with: -> email { email.strip.downcase }
  normalizes :phone, with: -> phone { phone.delete("^0-9").delete_prefix("1") }
end

user = User.create(email: " CRUISE-CONTROL@EXAMPLE.COM\n")
user.email                  # => "cruise-control@example.com"

user = User.find_by(email: "\tCRUISE-CONTROL@EXAMPLE.COM ")
user.email                  # => "cruise-control@example.com"
user.email_before_type_cast # => "cruise-control@example.com"

User.where(email: "\tCRUISE-CONTROL@EXAMPLE.COM ").count # => 1

User.exists?(email: "\tCRUISE-CONTROL@EXAMPLE.COM ")         # => true
User.exists?(["email = ?", "\tCRUISE-CONTROL@EXAMPLE.COM "]) # => false

User.normalize_value_for(:phone, "+1 (555) 867-5309") # => "5558675309"
```

### Add `ActiveRecord::Base.generates_token_for`

A new [method `generates_token_for`](https://github.com/rails/rails/pull/44189) has been introduced
to `ActiveRecord::Base`. This feature allows you to generate tokens that can embed data from a record.
These tokens are particularly useful for tasks like password resets.

With `generates_token_for`, tokens can be designed to reflect record state, making it possible to embed
specific record data within the token itself. When utilizing the token to retrieve the associated record,
a comparison is performed between the data in the token and the current data in the record. If the two
sets of data do not match, the token is considered invalid, similar to an expired token.

Here's an example of how this feature can be used:

```ruby
class User < ActiveRecord::Base
  has_secure_password

  generates_token_for :password_reset, expires_in: 15.minutes do
    # A password's BCrypt salt changes when the password is updated.
    # By embedding (part of) the salt in a token, the token will
    # expire when the password is updated.
    BCrypt::Password.new(password_digest).salt[-10..]
  end
end

user = User.first
token = user.generate_token_for(:password_reset)

User.find_by_token_for(:password_reset, token) # => user

user.update!(password: "new password")
User.find_by_token_for(:password_reset, token) # => nil
```

### Add `perform_all_later` to enqueue multiple jobs at once

The [`perform_all_later` method in ActiveJob](https://github.com/rails/rails/pull/46603),
designed to streamline the process of enqueuing multiple jobs simultaneously. This powerful
addition allows you to efficiently enqueue jobs without triggering callbacks. This is
particularly useful when you need to enqueue a batch of jobs at once, reducing the overhead
of multiple round-trips to the queue datastore.

Here's how you can take advantage of `perform_all_later`:

```ruby
# Enqueueing individual jobs
ActiveJob.perform_all_later(MyJob.new("hello", 42), MyJob.new("world", 0))

# Enqueueing an array of jobs
user_jobs = User.pluck(:id).map { |id| UserJob.new(user_id: id) }
ActiveJob.perform_all_later(user_jobs)
```

By utilizing `perform_all_later`, you can optimize your job enqueuing process and take advantage
of improved efficiency, especially when working with large sets of jobs. It's worth noting that
for queue adapters that support the new `enqueue_all` method, such as the Sidekiq adapter, the
enqueuing process is further optimized using `push_bulk`.

Please be aware that this new method introduces a separate event, `enqueue_all.active_job`,
and does not utilize the existing `enqueue.active_job` event. This ensures accurate tracking
and reporting of the bulk enqueuing process.

### Composite primary keys

TODO: Add description

### Introduce adapter for `Trilogy`

A [new adapter has been introduced](https://github.com/rails/rails/pull/47880) to facilitate the
seamless integration of `Trilogy`, a MySQL-compatible database client, with Rails applications.
Now, Rails applications have the option to incorporate `Trilogy` functionality by configuring their
`config/database.yml` file. For instance:

```yaml
development:
  adapter: trilogy
  database: blog_development
  pool: 5
```

Alternatively, integration can be achieved using the `DATABASE_URL` environment variable:

```ruby
ENV['DATABASE_URL'] # => "trilogy://localhost/blog_development?pool=5"
```

### Add `ActiveSupport::MessagePack`

TODO: Add description https://github.com/rails/rails/pull/47770

### Introducing `config.autoload_lib` and `config.autoload_lib_once` for Enhanced Autoloading

A [new configuration method, `config.autoload_lib(ignore:)`](https://github.com/rails/rails/pull/48572),
has been introduced. This method is used to enhance the autoload paths of applications by including the
`lib` directory, which is not included by default. Also, `config.autoload_lib(ignore: %w(assets tasks))`
is generated for new applications.

When invoked from either `config/application.rb` or `config/environments/*.rb`, this method adds the
`lib` directory to both `config.autoload_paths` and `config.eager_load_paths`. It's important to note
that this feature is not available for engines.

To ensure flexibility, the `ignore` keyword argument can be used to specify subdirectories within the
`lib` directory that should not be managed by the autoloaders. For instance, you can exclude directories
like `assets`, `tasks`, and `generators` by passing them to the `ignore` argument:

```ruby
config.autoload_lib(ignore: %w(assets tasks generators))
```

The [`config.autoload_lib_once` method](https://github.com/rails/rails/pull/48610) is similar to
`config.autoload_lib`, except that it adds `lib` to `config.autoload_once_paths` instead.

Please, see more details in the [autoloading guide](autoloading_and_reloading_constants.html#config-autoload-lib-ignore)

### Active Record API for general async queries

A significant enhancement has been introduced to the Active Record API, expanding its
[support for asynchronous queries](https://github.com/rails/rails/pull/44446). This enhancement
addresses the need for more efficient handling of not-so-fast queries, particularly focusing on
aggregates (such as `count`, `sum`, etc.) and all methods returning a single record or anything
other than a `Relation`.

The new API includes the following asynchronous methods:

- `async_count`
- `async_sum`
- `async_minimum`
- `async_maximum`
- `async_average`
- `async_pluck`
- `async_pick`
- `async_find_by_sql`
- `async_count_by_sql`

Here's a brief example of how to use one of these methods, `async_count`, to count the number of published
posts in an asynchronous manner:

```ruby
# Synchronous count
published_count = Post.where(published: true).count # => 10

# Asynchronous count
promise = Post.where(published: true).async_count # => #<ActiveRecord::Promise status=pending>
promise.value # => 10
```

These methods allow for the execution of these operations in an asynchronous manner, which can significantly
improve performance for certain types of database queries.

### Allow templates to set strict `locals`.

Introduce a new feature that [allows templates to set explicit `locals`](https://github.com/rails/rails/pull/45602).
This enhancement provides greater control and clarity when passing variables to your templates.

By default, templates will accept any `locals` as keyword arguments. However, now you can define what `locals` a
template should accept by adding a `locals` magic comment at the beginning of your template file.

Here's how it works:

```erb
<%# locals: (message:) -%>
<%= message %>
```

You can also set default values for these locals:

```erb
<%# locals: (message: "Hello, world!") -%>
<%= message %>
```

If you want to disable the use of locals entirely, you can do so like this:

```erb
<%# locals: () %>
```

### Add `Rails.application.deprecators`

The new [`Rails.application.deprecators` method](https://github.com/rails/rails/pull/46049) returns a
collection of managed deprecators within your application, and allows you to add and retrieve individual
deprecators with ease:

```ruby
Rails.application.deprecators[:my_gem] = ActiveSupport::Deprecation.new("2.0", "MyGem")
Rails.application.deprecators[:other_gem] = ActiveSupport::Deprecation.new("3.0", "OtherGem")
```

The collection's configuration settings affect all deprecators in the collection.

```ruby
Rails.application.deprecators.debug = true

Rails.application.deprecators[:my_gem].debug
# => true

Rails.application.deprecators[:other_gem].debug
# => true
```

There are scenarios where you might want to mute all deprecator warnings for a specific block of code.
With the deprecators collection, you can easily silence all deprecator warnings within a block:

```ruby
Rails.application.deprecators.silence do
  Rails.application.deprecators[:my_gem].warn    # No warning (silenced)
  Rails.application.deprecators[:other_gem].warn # No warning (silenced)
end
```

### Support pattern matching for JSON `response.parsed_body`

When `ActionDispatch::IntegrationTest` tests blocks invoke
`response.parsed_body` for JSON responses, their payloads will be available with
indifferent access. This enables integration with [Ruby's Pattern
Matching][pattern-matching], and built-in [Minitest support for pattern
matching][minitest-pattern-matching]:

```ruby
get "/posts.json"

response.content_type         # => "application/json; charset=utf-8"
response.parsed_body.class    # => Array
response.parsed_body          # => [{"id"=>42, "title"=>"Title"},...

assert_pattern { response.parsed_body => [{ id: 42 }] }

get "/posts/42.json"

response.content_type         # => "application/json; charset=utf-8"
response.parsed_body.class    # => ActiveSupport::HashWithIndifferentAccess
response.parsed_body          # => {"id"=>42, "title"=>"Title"}

assert_pattern { response.parsed_body => [{ title: /title/i }] }
```

[pattern-matching]: https://docs.ruby-lang.org/en/master/syntax/pattern_matching_rdoc.html
[minitest-pattern-matching]: https://docs.seattlerb.org/minitest/Minitest/Assertions.html#method-i-assert_pattern

### Extend `response.parsed_body` to parse HTML with Nokogiri

[Extend the `ActionDispatch::Testing` module][#47144] to support parsing the
value of an HTML `response.body` into a `Nokogiri::HTML5::Document` instance:

```ruby
get "/posts"

response.content_type         # => "text/html; charset=utf-8"
response.parsed_body.class    # => Nokogiri::HTML5::Document
response.parsed_body.to_html  # => "<!DOCTYPE html>\n<html>\n..."
```

Newly added [Nokogiri support for pattern matching][nokogiri-pattern-matching],
along with built-in [Minitest support for pattern
matching][minitest-pattern-matching] presents opportunities to make test
assertions about the structure and content of the HTML response:

```ruby
get "/posts"

html = response.parsed_body # => <html>
                            #      <head></head>
                            #        <body>
                            #          <main><h1>Some main content</h1></main>
                            #        </body>
                            #     </html>

assert_pattern { html.at("main") => { content: "Some main content" } }
assert_pattern { html.at("main") => { content: /content/ } }
assert_pattern { html.at("main") => { children: [{ name: "h1", content: /content/ }] } }
```

[#47144]: https://github.com/rails/rails/pull/47144
[nokogiri-pattern-matching]: https://nokogiri.org/rdoc/Nokogiri/XML/Attr.html#method-i-deconstruct_keys
[minitest-pattern-matching]: https://docs.seattlerb.org/minitest/Minitest/Assertions.html#method-i-assert_pattern

Railties
--------

Please refer to the [Changelog][railties] for detailed changes.

### Removals

*   Remove deprecated `bin/rails secrets:setup` command.

*   Remove default `X-Download-Options` header since it is used only by Internet Explorer.

### Deprecations

*   Deprecated usage of `Rails.application.secrets`.

*   Deprecated `secrets:show` and `secrets:edit` commands in favor of `credentials`.

*   Deprecated `Rails::Generators::Testing::Behaviour` in favor of `Rails::Generators::Testing::Behavior`.

### Notable changes

*   Add `sandbox_by_default` option to start rails console in sandbox mode by default.

*   Add new syntax for support filtering tests by line ranges.

*   Add `DATABASE` option that enables the specification of the target database when executing the
    `rails railties:install:migrations` command to copy migrations.

Action Cable
------------

Please refer to the [Changelog][action-cable] for detailed changes.

### Removals

### Deprecations

### Notable changes

*   Add `capture_broadcasts` test helper to capture all messages broadcasted in a block.

*   Add the ability to Redis pub/sub adapter to automatically reconnect when Redis connection is lost.

*   Add command callbacks `before_command`, `after_command`, and `around_command` to `ActionCable::Connection::Base`.

Action Pack
-----------

Please refer to the [Changelog][action-pack] for detailed changes.

### Removals

*   Remove deprecated behavior on `Request#content_type`

*   Remove deprecated ability to assign a single value to `config.action_dispatch.trusted_proxies`.

*   Remove deprecated `poltergeist` and `webkit` (capybara-webkit) driver registration for system testing.

### Deprecations

*   Deprecate `config.action_dispatch.return_only_request_media_type_on_content_type`.

*   Deprecate `AbstractController::Helpers::MissingHelperError`

*   Deprecate `ActionDispatch::IllegalStateError`.

*   Deprecate `speaker`, `vibrate`, and `vr` permissions policy directives.

### Notable changes

*   Add `exclude?` method to `ActionController::Parameters`. It is the inverse of `include?` method.

*   Add `ActionController::Parameters#extract_value` method to allow extracting serialized values from params.

*   Add the ability to use custom logic for storing and retrieving CSRF tokens.

*   Add `html` and `screenshot` kwargs for system test screenshot helper.

Action View
-----------

Please refer to the [Changelog][action-view] for detailed changes.

### Removals

*   Remove deprecated constant `ActionView::Path`.

*   Remove deprecated support to passing instance variables as locals to partials.

### Deprecations

### Notable changes

*   `check_box_tag` and `radio_button_tag` now accept `checked` as a keyword argument.

*   Add `picture_tag` helper to generate HTML `<picture>` tags.

*   The `simple_format` helper now handles a `:sanitize_options` feature, allowing the addition of extra
    options for the sanitize process.

    ```ruby
    simple_format("<a target=\"_blank\" href=\"http://example.com\">Continue</a>", {}, { sanitize_options: { attributes: %w[target href] } })
    # => "<p><a target=\"_blank\" href=\"http://example.com\">Continue</a></p>"
    ```

Action Mailer
-------------

Please refer to the [Changelog][action-mailer] for detailed changes.

### Removals

### Deprecations

*   Deprecated `config.action_mailer.preview_path`.

*   Deprecated passing params to `assert_enqueued_email_with` via the `:args` kwarg.
    Now supports a `:params` kwarg, so use that to pass params.

### Notable changes

*   Add `config.action_mailer.preview_paths` to support multiple preview paths.

*   Add `capture_emails` in the test helper to capture all emails sent in a block.

*   Add `deliver_enqueued_emails` to `ActionMailer::TestHelper` to deliver all enqueued email jobs.


Active Record
-------------

Please refer to the [Changelog][active-record] for detailed changes.

### Removals

*   Remove support for `ActiveRecord.legacy_connection_handling`.

*   Remove deprecated `ActiveRecord::Base` config accessors

*   Remove support for `:include_replicas` on `configs_for`. Use `:include_hidden` instead.

*   Remove deprecated `config.active_record.partial_writes`.

*   Remove deprecated `Tasks::DatabaseTasks.schema_file_type`.

*   Remove `--no-comments` flag in structure dumps for PostgreSQL.

### Deprecations

*   Deprecate `name` argument on `#remove_connection`.

*   Deprecate `check_pending!` in favor of `check_all_pending!`.

*   Deprecate `deferrable: true` option of `add_foreign_key` in favor of `deferrable: :immediate`.

*   Deprecate `TestFixtures#fixture_path` in favor of `TestFixtures#fixture_paths`.

*   Deprecate delegation from `Base` to `connection_handler`.

*   Deprecate `config.active_record.suppress_multiple_database_warning`.

*   Deprecate using `ActiveSupport::Duration` as an interpolated bind parameter in a SQL
    string template.

*   Deprecate `all_connection_pools` and make `connection_pool_list` more explicit.

*   Deprecate `read_attribute(:id)` returning the primary key if the primary key is not `:id`.

*   Deprecate `rewhere` argument on `#merge`.

### Notable changes

*   Add `TestFixtures#fixture_paths` to support multiple fixture paths.

Active Storage
--------------

Please refer to the [Changelog][active-storage] for detailed changes.

### Removals

*   Remove deprecated invalid default content types in Active Storage configurations.

*   Remove deprecated `ActiveStorage::Current#host` and `ActiveStorage::Current#host=` methods.

*   Remove deprecated behavior when assigning to a collection of attachments. Instead of appending to the collection,
    the collection is now replaced.

*   Remove deprecated `purge` and `purge_later` methods from the attachments association.

### Deprecations

### Notable changes

*   `ActiveStorage::Analyzer::AudioAnalyzer` now outputs `sample_rate` and `tags` in the output `metadata` hash.

*   Add the option to utilize predefined variants when invoking the `preview` or `representation` methods on an
    attachment.

*   `preprocessed` option is added when declaring variants to preprocess variants.

*   Add the ability to destroy active storage variants.

    ```ruby
    User.first.avatar.variant(resize_to_limit: [100, 100]).destroy
    ```

Active Model
------------

Please refer to the [Changelog][active-model] for detailed changes.

### Removals

### Deprecations

### Notable changes

*   Add support for infinite ranges to `LengthValidator`s `:in`/`:within` options.

    ```ruby
    validates_length_of :first_name, in: ..30
    ```

*   Add support for beginless ranges to `inclusivity/exclusivity` validators.

     ```ruby
    validates_inclusion_of :birth_date, in: -> { (..Date.today) }
    ```

*   Add support for password challenges to `has_secure_password`. When set, validate that the password
    challenge matches the persisted `password_digest`.

*   Allow validators to accept lambdas without record argument.

    ```ruby
    # Before
    validates_comparison_of :birth_date, less_than_or_equal_to: ->(_record) { Date.today }

    # After
    validates_comparison_of :birth_date, less_than_or_equal_to: -> { Date.today }
    ```

Active Support
--------------

Please refer to the [Changelog][active-support] for detailed changes.

### Removals

*   Remove deprecated override of `Enumerable#sum`.

*   Remove deprecated `ActiveSupport::PerThreadRegistry`.

*   Remove deprecated option to passing a format to `#to_s` in `Array`, `Range`, `Date`, `DateTime`, `Time`,
    `BigDecimal`, `Float` and, `Integer`.

*   Remove deprecated override of `ActiveSupport::TimeWithZone.name`.

*   Remove deprecated `active_support/core_ext/uri` file.

*   Remove deprecated `active_support/core_ext/range/include_time_with_zone` file.

*   Remove implicit conversion of objects into `String` by `ActiveSupport::SafeBuffer`.

*   Remove deprecated support to generate incorrect RFC 4122 UUIDs when providing a namespace ID that is not one of the
    constants defined on `Digest::UUID`.

### Deprecations

*   Deprecate `config.active_support.disable_to_s_conversion`.

*   Deprecate `config.active_support.remove_deprecated_time_with_zone_name`.

*   Deprecate `config.active_support.use_rfc4122_namespaced_uuids`.

*   Deprecate `SafeBuffer#clone_empty`.

*   Deprecate usage of the singleton `ActiveSupport::Deprecation`.

*   Deprecate initializing a `ActiveSupport::Cache::MemCacheStore` with an instance of `Dalli::Client`.

*   Deprecate `Notification::Event`'s `#children` and `#parent_of?` methods.

### Notable changes

Active Job
----------

Please refer to the [Changelog][active-job] for detailed changes.

### Removals

*   Remove `QueAdapter`.

### Deprecations

### Notable changes

*   Add `perform_all_later` to enqueue multiple jobs at once.

*   Add `--parent` option to job generator to specify parent class of job.

*   Add `after_discard` method to `ActiveJob::Base` to run a callback when a job is about to be discarded.

*   Add support for logging background job enqueue callers.

Action Text
----------

Please refer to the [Changelog][action-text] for detailed changes.

### Removals

### Deprecations

### Notable changes

Action Mailbox
----------

Please refer to the [Changelog][action-mailbox] for detailed changes.

### Removals

### Deprecations

### Notable changes

*   Add `X-Forwarded-To` addresses to recipients.

*   Add `bounce_now_with` method to `ActionMailbox::Base` to send the bounce email without going through a
    mailer queue.

Ruby on Rails Guides
--------------------

Please refer to the [Changelog][guides] for detailed changes.

### Notable changes

Credits
-------

See the
[full list of contributors to Rails](https://contributors.rubyonrails.org/)
for the many people who spent many hours making Rails, the stable and robust
framework it is. Kudos to all of them.

[railties]:       https://github.com/rails/rails/blob/main/railties/CHANGELOG.md
[action-pack]:    https://github.com/rails/rails/blob/main/actionpack/CHANGELOG.md
[action-view]:    https://github.com/rails/rails/blob/main/actionview/CHANGELOG.md
[action-mailer]:  https://github.com/rails/rails/blob/main/actionmailer/CHANGELOG.md
[action-cable]:   https://github.com/rails/rails/blob/main/actioncable/CHANGELOG.md
[active-record]:  https://github.com/rails/rails/blob/main/activerecord/CHANGELOG.md
[active-storage]: https://github.com/rails/rails/blob/main/activestorage/CHANGELOG.md
[active-model]:   https://github.com/rails/rails/blob/main/activemodel/CHANGELOG.md
[active-support]: https://github.com/rails/rails/blob/main/activesupport/CHANGELOG.md
[active-job]:     https://github.com/rails/rails/blob/main/activejob/CHANGELOG.md
[action-text]:    https://github.com/rails/rails/blob/main/actiontext/CHANGELOG.md
[action-mailbox]: https://github.com/rails/rails/blob/main/actionmailbox/CHANGELOG.md
[guides]:         https://github.com/rails/rails/blob/main/guides/CHANGELOG.md
