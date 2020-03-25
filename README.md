[![Cult Of Martians](http://cultofmartians.com/assets/badges/badge.svg)](http://cultofmartians.com)
[![Gem Version](https://badge.fury.io/rb/anyway_config.svg)](https://rubygems.org/gems/anyway_config) [![Build](https://github.com/palkan/anyway_config/workflows/Build/badge.svg)](https://github.com/palkan/anyway_config/actions)
[![JRuby Build](https://github.com/palkan/anyway_config/workflows/JRuby%20Build/badge.svg)](https://github.com/palkan/anyway_config/actions)

# Anyway Config

> One configuration to rule all data sources

Anyway Config is a configuration library for Ruby gems and applications.

As a library author, you can benefit from using Anyway Config by providing a better UX for your end-users:

- **Zero-code configuration** — no more boilerplate initializers.
- **Per-environment and local** settings support out-of-the-box.

For application developers, Anyway Config could be useful to:

- **Keep configuration organized** and use _named configs_ instead of bloated `.env`/`settings.yml`/whatever.
- **Free code of ENV/credentials/secrets dependency** and use configuration classes instead—your code should not rely on configuration data sources.

**NOTE:** this readme shows documentation for the upcoming 2.0 version (`2.0.0.pre2` is available on RubyGems).
For version 1.x see [1-4-stable branch](https://github.com/palkan/anyway_config/tree/1-4-stable).

## Table of contents

- [Main concepts](#main-concepts)
- [Installation](#installation)
- [Usage](#usage)
  - [Configuration classes](#configuration-classes)
  - [Dynamic configuration](#dynamic-configuration)
  - [Validation](#validation)
- [Using with Rails applications](#using-with-rails)
  - [Data population](#data-population)
  - [Organizing configs](#organizing-configs)
  - [Generators](#generators)
- [Using with Ruby applications](#using-with-ruby)
- [Environment variables](#environment-variables)
- [Local configuration](#local-files)
- [Data loaders](#data-loaders)
- [Source tracing](#tracing)
- [Test helpers](#test-helpers)
- [OptionParser integration](#optionparser-integration)

## Main concepts

Anyway Config abstractize the configuration layer by introducing **configuration classes** which describe available parameters and their defaults. For [example](https://github.com/palkan/influxer/blob/master/lib/influxer/config.rb):

```ruby
module Influxer
  class Config < Anyway::Config
    attr_config(
      host: "localhost",
      username: "root",
      password: "root"
    )
  end
end
```

Using Ruby classes to represent configuration allows you to add helper methods and computed parameters easily, makes the configuration **testable**.

The `anyway_config` gem takes care of loading parameters from **different sources** (YAML, credentials/secrets, environment variables, etc.). Internally, we use a _pipeline pattern_ and provide the [Loaders API](#data-loaders) to manage and [extend](#custom-loaders) its functionality.

Check out the libraries using Anyway Config for more examples:

- [Influxer](https://github.com/palkan/influxer)
- [AnyCable](https://github.com/anycable/anycable)
- [Sniffer](https://github.com/aderyabin/sniffer)
- [Blood Contracts](https://github.com/sclinede/blood_contracts)
- [and others](https://github.com/palkan/anyway_config/network/dependents).

## Installation

Adding to a gem:

```ruby
# my-cool-gem.gemspec
Gem::Specification.new do |spec|
  # ...
  spec.add_dependency "anyway_config", "2.0.0.pre2"
  # ...
end
```

Or adding to your project:

```ruby
# Gemfile
gem "anyway_config", "2.0.0.pre2"
```

### Supported Ruby versions

- Ruby (MRI) >= 2.5.0
- JRuby >= 9.2.9

## Usage

### Configuration classes

Using configuration classes allows you to make configuration data a bit more than a bag of values:
you can define a schema for your configuration, provide defaults, add validations and additional helper methods.

Anyway Config provides a base class to inherit from with a few DSL methods:

```ruby
require "anyway_config"

module MyCoolGem
  class Config < Anyway::Config
    attr_config user: "root", password: "root", host: "localhost"
  end
end
```

Here `attr_config` creates accessors and populates the default values. If you don't need default values you can write:

```ruby
attr_config :user, :password, host: "localhost", options: {}
```

**NOTE**: it's safe to use non-primitive default values (like Hashes or Arrays) without worrying about their mutation: the values would be deeply duplicated for each config instance.

Then, create an instance of the config class and use it:

```ruby
MyCoolGem::Config.new.user #=> "root"
```

**NOTE**: since v2.0 accessors created by `attr_config` are not `attr_accessor`, i.e. they do not populate instance variables. If you used instance variables before to override readers, you must switch to using `super` or `values` store:

```ruby
class MyConfig < Anyway::Config
  attr_config :host, :port, :url, :meta

  # override writer to handle type coercion
  def meta=(val)
    super JSON.parse(val)
  end

  # or override reader to handle missing values
  def url
    values[:url] ||= "#{host}:#{port}"
  end

  # untill v2.1, it will still be possible to read instance variables,
  # i.e. the following code would also work
  def url
    @url ||= "#{host}:#{port}"
  end
end
```

#### Config name

Anyway Config relies on the notion of _config name_ to populate data.

By default, Anyway Config uses the config class name to infer the config name using the following rules:

- if the class name has a form of `<Module>::Config` then use the module name (`SomeModule::Config => "somemodule"`)
- if the class name has a form of `<Something>Config` then use the class name prefix (`SomeConfig => "some"`)

**NOTE:** in both cases, the config name is a **downcased** module/class prefix, not underscored.

You can also specify the config name explicitly (it's required in cases when your class name doesn't match any of the patterns above):

```ruby
module MyCoolGem
  class Config < Anyway::Config
    config_name :cool
    attr_config user: "root", password: "root", host: "localhost", options: {}
  end
end
```

#### Customize env variable names prefix

By default, Anyway Config uses upper-cased config name as a prefix for env variable names (e.g.
`config_name :my_app` will result to parsing `MY_APP_` prefix).

You can set env prefix explicitly:

```ruby
module MyCoolGem
  class Config < Anyway::Config
    config_name :cool_gem
    env_prefix :really_cool # now variables, starting wih `REALLY_COOL_`, will be parsed
    attr_config user: "root", password: "root", host: "localhost", options: {}
  end
end
```

#### Explicit values

Sometimes it's useful to set some parameters explicitly during config initialization.
You can do that by passing a Hash into `.new` method:

```ruby
config = MyCoolGem::Config.new(
  user: "john",
  password: "rubyisnotdead"
)

# The value would not be overridden from other sources (such as YML file, env)
config.user == "john"
```

#### Reload configuration

There are `#clear` and `#reload` methods that do exactly what they state.

**NOTE**: `#reload` also accepts an optional Hash for [explicit values](#explicit-values).

### Dynamic configuration

You can also fetch configuration without pre-defined schema:

```ruby
# load data from config/my_app.yml,
# credentials.my_app, secrets.my_app (if using Rails), ENV["MY_APP_*"]
#
# Given MY_APP_VALUE=42
config = Anyway::Config.for(:my_app)
config["value"] #=> 42

# you can specify the config file path or env prefix
config = Anyway::Config.for(:my_app, config_path: "my_config.yml", env_prefix: "MYAPP")
```

This feature is similar to `Rails.application.config_for` but more powerful:

| Feature | Rails | Anyway Config |
| ------------- |-------------:| -----:|
| Load data from `config/app.yml` | ✅ | ✅ |
| Load data from `secrets` | ❌ | ✅ |
| Load data from `credentials` | ❌ | ✅ |
| Load data from environment | ❌ | ✅ |
| Load data from [custom sources](#data-loaders) | ❌ | ✅ |
| Local config files | ❌ | ✅ |
| [Source tracing](#tracing) | ❌ | ✅ |
| Return Hash with indifferent access | ❌ | ✅ |
| Support ERB\* within `config/app.yml` | ✅ | ✅ |
| Raise if file doesn't exist | ✅ | ❌ |
| Works without Rails | 😀 | ✅ |

\* Make sure that ERB is loaded

### Validation

Anyway Config provides basic ways of ensuring that the configuration is valid.

There is a built-in `required` class method to define the list of parameters that must be present in the
configuration after loading (where present means non-`nil` and non-empty for strings):

```ruby
class MyConfig < Anyway::Config
  attr_config :api_key, :api_secret, :debug

  required :api_key, :api_secret
end

MyConfig.new(api_secret: "") #=> raises Anyway::Config::ValidationError
```

If you need more flexibility, you can override the `#validate!` method in your configuration class:

```ruby
class MyConfig < Anyway::Config
  attr_config :api_key, :api_secret, :mode

  def validate!
    # don't forget to call super if you're using `required`
    super
    unless %w[production test].include?(mode)
      raise_validation_error "Unknown mode; #{mode}"
    end
  end
end
```

## Using with Rails

**NOTE:** version 2.x supports Rails >= 5.0; for Rails 4.x use version 1.x of the gem.

We recommend going through [Data population](#data-population) and [Organizing configs](#organizing-configs) sections first,
and then use [Rails generators](#generators) to make your application Anyway Config-ready.

### Data population

Your config is filled up with values from the following sources (ordered by priority from low to high):

- `RAILS_ROOT/config/my_cool_gem.yml` (for the current `RAILS_ENV`, supports `ERB`):

```yml
test:
  host: localhost
  port: 3002

development:
  host: localhost
  port: 3000
```

**NOTE:** you can override the default YML lookup path by setting `MYCOOLGEM_CONF` env variable.

- `Rails.application.secrets.my_cool_gem` (if `secrets.yml` present):

```yml
# config/secrets.yml
development:
  my_cool_gem:
    port: 4444
```

- `Rails.application.credentials.my_cool_gem` (if supported):

```yml
my_cool_gem:
  host: secret.host
```

**NOTE:** You can backport Rails 6 per-environment credentials to Rails 5.2 app using [this patch](https://gist.github.com/palkan/e27e4885535ff25753aefce45378e0cb).

- `ENV['MYCOOLGEM_*']`.

See [environment variables](#environment-variables).

### Organizing configs

You can store application-level config classes in `app/configs` folder just like any other Rails entities.

However, in that case you won't be able to use them during the application initialization (i.e., in `config/**/*.rb` files).

Since that's a pretty common scenario, we provide a way to do that via a custom autoloader for `config/configs` folder.
That means, that you can put your configuration classes into `config/configs` folder, use them anywhere in your code without explicitly requiring them.

Consider an example: setting the Action Mailer hostname for Heroku review apps.

We have the following config to fetch the Heroku provided [metadata](https://devcenter.heroku.com/articles/dyno-metadata):

```ruby
# This data is provided by Heroku Dyno Metadadata add-on.
class HerokuConfig < Anyway::Config
  attr_config :app_id, :app_name,
    :dyno_id, :release_version,
    :slug_commit

  def hostname
    "#{app_name}.herokuapp.com"
  end
end
```

Then in `config/application.rb` you can do the following:

```ruby
config.action_mailer.default_url_options = {host: HerokuConfig.new.hostname}
```

You can configure the configs folder path:

```ruby
# The path must be relative to Rails root
config.anyway_config.autoload_static_config_path = "path/to/configs"
```

**NOTE:** Configs loaded from the `autoload_static_config_path` are **not reloaded in development**. We call them _static_. So, it makes sense to keep only configs necessary for initialization in this folder. Other configs, _dynamic_, could be stored in `app/configs`.
Or you can store everything in `app/configs` by setting `config.anyway_config.autoload_static_config_path = "app/configs"`.

### Generators

Anyway Config provides Rails generators to create new config classes:

- `rails g anyway:install`—creates an `ApplicationConfig` class (the base class for all config classes) and updates `.gitignore`

You can specify the static configs path via the `--configs-path` option:

```sh
rails g anyway:install --configs-path=config/settings

# or to keep everything in app/configs
rails g anyway:install --configs-path=app/configs
```

- `rails g anyway:config <name> param1 param2 ...`—creates a named configuration class and optionally the corresponding YAML file; creates `application_config.rb` is missing.

The generator command for the Heroku example above would be:

```sh
$ rails g anyway:config heroku app_id app_name dyno_id release_version slug_commit

    generate  anyway:install
       rails  generate anyway:install
      create  config/configs/application_config.rb
      append  .gitignore
      create  config/configs/heroku_config.rb
Would you like to generate a heroku.yml file? (Y/n) n
```

You can also specify the `--app` option to put the newly created class into `app/configs` folder.
Alternatively, you can call `rails g anyway:app_config name param1 param2 ...`.

## Using with Ruby

The default data loading mechanism for non-Rails applications is the following (ordered by priority from low to high):

- `./config/<config-name>.yml` (`ERB` is supported if `erb` is loaded)

In pure Ruby apps, we do not know about _environments_ (`test`, `development`, `production`, etc.); thus, we assume that the YAML contains values for a single environment:

```yml
host: localhost
port: 3000
```

**NOTE:** you can override the default YML lookup path by setting `MYCOOLGEM_CONF` env variable.

- `ENV['MYCOOLGEM_*']`.

See [environment variables](#environment-variables).

## Environment variables

Environmental variables for your config should start with your config name, upper-cased.

For example, if your config name is "mycoolgem", then the env var "MYCOOLGEM_PASSWORD" is used as `config.password`.

Environment variables are automatically type cast:

- `"True"`, `"t"` and `"yes"` to `true`;
- `"False"`, `"f"` and `"no"` to `false`;
- `"nil"` and `"null"` to `nil` (do you really need it?);
- `"123"` to 123 and `"3.14"` to 3.14.

*Anyway Config* supports nested (_hashed_) env variables—just separate keys with double-underscore.

For example, "MYCOOLGEM_OPTIONS__VERBOSE" is parsed as `config.options["verbose"]`.

Array values are also supported:

```ruby
# Suppose ENV["MYCOOLGEM_IDS"] = '1,2,3'
config.ids #=> [1,2,3]
```

If you want to provide a text-like env variable which contains commas then wrap it into quotes:

```ruby
MYCOOLGEM = "Nif-Nif, Naf-Naf and Nouf-Nouf"
```

## Local files

It's useful to have a personal, user-specific configuration in development, which extends the project-wide one.

We support this by looking at _local_ files when loading the configuration data:

- `<config_name>.local.yml` files (next to\* the _global_ `<config_name>.yml`)
- `config/credentials/local.yml.enc` (for Rails >= 6, generate it via `rails credentials:edit --environment local`).

\* If the YAML config path is not a default one (i.e., set via `<CONFIG_NAME>_CONF`), we look up the local
config at this location, too.

Local configs are meant for using in development and only loaded if `Anyway::Settings.use_local_files` is `true` (which is true by default if `RACK_ENV` or `RAILS_ENV` env variable is equal to `"development"`).

**NOTE:** in Rails apps you can use `Rails.application.configuration.anyway_config.use_local_files`.

Don't forget to add `*.local.yml` (and `config/credentials/local.*`) to your `.gitignore`.

**NOTE:** local YAML configs for a Rails app must be environment-free (i.e., you shouldn't have top-level `development:` key).

## Data loaders

You can provide your own data loaders or change the existing ones using the Loaders API (which is very similar to Rack middleware builder):

```ruby
# remove env loader => do not load params from ENV
Anyway.loaders.delete :env

# add custom loader before :env (it's better to keep the ENV loader the last one)
Anyway.loaders.insert_before :env, :my_loader, MyLoader
```

Loader is a _callable_ Ruby object (module/class responding to `.call` or lambda/proc), which `call` method
accepts the following keyword arguments:

```ruby
def call(
  name:, # config name
  env_prefix:, # prefix for env vars if any
  config_path:, # path to YML config
  local: # true|false, whether to load local configuration
)
  #=> must return Hash with configuration data
end
```

You can use `Anyway::Loaders::Base` as a base class for your loader and define a `#call` method.
For example, the [Chamber](https://github.com/thekompanee/chamber) loader could be written as follows:

```ruby
class ChamberConfigLoader < Anyway::Loaders::Base
  def call(name:, **_opts)
    Chamber.env.to_h[name] || {}
  end
end
```

## Tracing

Since Anyway Config loads data from multiple source, it could be useful to know where a particular value came from.

Each `Anyway::Config` instance contains _tracing information_ which you can access via `#to_source_trace` method:

```ruby
conf = ExampleConfig.new
conf.to_source_trace

# returns the following hash
{
  "host" => {value: "test.host", source: {type: :yml, path: "config/example.yml"}},
  "user" => {
    "name" => {value: "john", source: {type: :env, key: "EXAMPLE_USER__NAME"}},
    "password" => {value: "root", source: {type: :credentials, store: "config/credentials/production.enc.yml"}}
  },
  "port" => {value: 9292, source: {type: :defaults}}
}

# if you change the value manually in your code,
# that would be reflected in the trace

conf.host = "anyway.host"
conf.to_source_trace["host"]
#=> {type: :user, called_from: "/path/to/caller.rb:15"}
```

## Test helpers

We provide the `with_env` test helper to test code in the context of the specified environment variables values:

```ruby
describe HerokuConfig, type: :config do
  subject { described_class.new }

  specify do
    # Ensure that the env vars are set to the specified
    # values within the block and reset to the previous values
    # outside of it.
    with_env(
      "HEROKU_APP_NAME" => "kin-web-staging",
      "HEROKU_APP_ID" => "abc123",
      "HEROKU_DYNO_ID" => "ddyy",
      "HEROKU_RELEASE_VERSION" => "v0",
      "HEROKU_SLUG_COMMIT" => "3e4d5a"
    ) do
      is_expected.to have_attributes(
        app_name: "kin-web-staging",
        app_id: "abc123",
        dyno_id: "ddyy",
        release_version: "v0",
        slug_commit: "3e4d5a"
      )
    end
  end
end
```

If you want to delete the env var, pass `nil` as the value.

This helper is automatically included to RSpec if `RAILS_ENV` or `RACK_ENV` env variable is equal to "test". It's only available for the example with the tag `type: :config` or with the path `spec/configs/...`.

You can add it manually by requiring `"anyway/testing/helpers"` and including the `Anyway::Test::Helpers` module (into RSpec configuration or Minitest test class).

## OptionParser integration

It's possible to use config as option parser (e.g., for CLI apps/libraries). It uses
[`optparse`](https://ruby-doc.org/stdlib-2.5.1/libdoc/optparse/rdoc/OptionParser.html) under the hood.

Example usage:

```ruby
class MyConfig < Anyway::Config
  attr_config :host, :log_level, :concurrency, :debug, server_args: {}

  # specify which options shouldn't be handled by option parser
  ignore_options :server_args

  # provide description for options
  describe_options(
    concurrency: "number of threads to use"
  )

  # mark some options as flag
  flag_options :debug

  # extend an option parser object (i.e. add banner or version/help handlers)
  extend_options do |parser, config|
    parser.banner = "mycli [options]"

    parser.on("--server-args VALUE") do |value|
      config.server_args = JSON.parse(value)
    end

    parser.on_tail "-h", "--help" do
      puts parser
    end
  end
end

config = MyConfig.new

config.parse_options!(%w[--host localhost --port 3333 --log-level debug])

config.host # => "localhost"
config.port # => 3333
config.log_level # => "debug"

# Get the instance of OptionParser
config.option_parser
```

**NOTE:** values are automatically type cast using the same rules as for [environment variables](#environment-variables).
If you want to specify the type explicitly, you can do that using `describe_options`:

```ruby
describe_options(
  # In this case, you should specify a hash with `type`
  # and (optionally) `desc` keys
  concurrency: {
    desc: "number of threads to use",
    type: String
  }
)
```

## Contributing

Bug reports and pull requests are welcome on GitHub at [https://github.com/palkan/anyway_config](https://github.com/palkan/anyway_config).

## License

The gem is available as open source under the terms of the [MIT License](http://opensource.org/licenses/MIT).
