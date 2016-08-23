# Ruby style guide

This guide builds upon [GitHub's Ruby style guide](https://github.com/styleguide/ruby).

## General

* Prefer the standard library over a gem unless there's a good reason.

## Ordering

Whenever there's a list of things, put one on each line, and sort the list
alphanumerically.

```ruby
# requires
require "aardvark/collection"
require "aardvark/record"
require "aardvark/reporter"
require "active_record"

# hashes
request_context = {
  :host => host,
  :port => port,
  :request_id => request.id,
  :user => @user,
  :user_name => @user.login
}

# accessors, validations, etc
attr_accessor :some_attribute
attr_accessor :some_other_attribute
```

## Classes

When things are in the same place for every class, it's easy to jump to find
what you expect.

```ruby
require "some/dependency"  # requires at the top

# Describe what this class is for, unless it's obvious from the name
class Foo
  # module decorations
  include Bar
  prepend Baz
  extend Garply

  # constants
  MY_REGEX = /foo/.freeze

  # dsl methods from modules
  some_bar_dsl_method
  validate :some_attribute

  # accessors
  attr_accessor :bar
  attr_accessor :foo

  # class methods
  def self.find(id)
  end

  # instance methods
  def initialize
  end

  private  # private methods at the bottom

  def private_method
  end
end
```

## Avoid returning in the middle of a method

Early returns are good for checking arguments. The implicit return for the last
expression of a method is also unsurprising. Returns in the middle of a method
makes it hard to reason what code has run, and often indicates a method has too
many responsibilities.

```ruby
def bad_method(args)
  intermediate_result = process(args)

  # bad
  return if intermediate_result.nil?

  intermediate_result.do_something
end

def good_method(args)
  # good
  return if args.nil? || args[:foo]

  process(args)
end
```

## Verbose is better than clever

* Avoid DSL's
* Avoid calling `send`
* Avoid `method_missing`


```ruby
class SomeModel
  # bad
  attribute :foo, :bar, :baz

  # good
  attr_accessor :bar
  attr_accessor :baz
  attr_accessor :foo
end
```

## Predicate methods returns booleans

```ruby
# bad, truthy value, not an actual boolean
def has_user?
  @user
end

# good
def has_user?
  !@user.nil?
end
```
