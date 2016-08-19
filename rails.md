# Ruby on Rails style guide

## Ruby

### Do not align by =, => delimiters

These introduce unnecessary diffs and require extra work and diffs to maintain over time

```ruby
# bad: adding a longer line requires realigning all other lines
assert_authz :get, @pub_owner_collab_url do |check|
  check.authorized_users           @owner, collaborator
  check.unauthorized_users         @rando
  check.authorized_installations   authorized_installation
  check.unauthorized_installations unauthorized_installation
  check.challenges                 :anon
end

# good
assert_authz :get, @pub_owner_collab_url do |check|
  check.authorized_users @owner, collaborator
  check.unauthorized_users @rando
  check.authorized_installations authorized_installation
  check.unauthorized_installations unauthorized_installation
  check.challenges :anon
end
```

## ActiveRecord

### Put validations on separate lines

This makes it easy to see what validations are on an attribute

```ruby
# bad
class User < ActiveRecord::Base
  validates :login, :email, :public_key, presence: true
  validates :email, format: { with: EMAIL_REGEX }
  validates :login, minimum: 2
end

# good
class User < ActiveRecord::Base
  validates :login, presence: true, minimum: 2
  validates :email, format: { with: EMAIL_REGEX }
  validates :public_key, presence: true
end
```

### Prefix validation methods with `validate_`

```ruby
class User < ActiveRecord::Base
  validate :validate_billing_information

  private

  def validate_billing_information
  end
end
```

### Group callbacks by type

```ruby
class User < ActiveRecord::Base
  before_validate :normalize_email
  before_create :deliver_welcome_email
  before_save :update_height
  before_save :update_billing_information
  before_destroy :verify_not_on_organizations
end
```

## Controllers

### Subclass and override instead of skipping filters

```ruby
class ApplicationController
  before_filter :require_login, :if => :login_required?

  private

  # Private: Override this in a subclass to customize whether login is required
  def login_required?
    true
  end
end
```

## Tests

### Use minitest style test methods

When a test fails, it's easy to grep for the original failing method name.

```ruby
# bad
context "User" do
  test "can't have more than one address" do
  end
end

# good
class UserTest < Minitest::Test
  def test_cannot_have_more_than_one_address
  end
end
```

### Clarity over Don't Repeat Yourself (DRY)

A test should be understandable without jumping through several methods and files.
DRY can be harmful in tests because it obscures the intention of the test.

```ruby
# bad: Custom test classes instead of vanilla test class
require "test/test_helpers/model_test"
class UserTest < ModelTest
  # bad: custom DSL hides how objects are setup
  subject User

  def test_validates_email
    # bad: what is @subject? What is it's login?
    assert_equal "#{@subject.login}@gmail.com", @subject.email
  end
end

# good
class UserTest < Minitest::Test
  # reading this method in isolation gives a good idea of what is being tested
  def test_validates_email
    user = User.new(login: "jch")

    # no string interpolation makes expected value obvious
    assert_equal "jch@gmail.com", @subject.email
  end
end
```


### Avoid nested contexts

This is a code smell that your class is doing too much. Group similar tests
together by names rather than nested blocks. This makes it easier to grep for a
test method, and minimizes nesting depth.

```ruby
# bad
class UserTest < Minitest::Test
  context "validations" do
    def test_requires_email
    end

    def test_requires_login
    end
  end
end

# good
class UserTest < Minitest::Test
  def test_validate_requires_email
  end

  def test_validate_requires_login
  end
end
```

### Avoid factory libraries

Objects that are difficult to instantiate in test are a code smell. Don't hide
them with a factory library.

```ruby
# bad
# factory_girl or machinist or some factory gem
User.blueprint do
  login
  email
  password { "foobar" }
end

def test_user_is_valid
  user = User.make
  assert_predicate user, :valid?
end

# good
def test_user_is_valid
  user = User.new(login: "foo", email: "foo@bar.com", password: "foobar")
  assert_predicate user, :valid?
end
```

### Avoid assert_difference and testing Model.last

Instead of testing that a record was created, test the results for what you care
about to avoid a brittle test coupled to your implementation.

```ruby
# bad. This test fails if creating a user generates more than one notification,
# even if the implementation still works.
def test_create_user_sends_a_notification_to_an_admin
  assert_difference "Notification.count" do
    some_action_that_creates_a_user
  end

  assert_equal "New user 'pizza' created", Notification.last.message
end

# good
def test_create_user_sends_a_notification_to_an_admin
  result = some_action_that_creates_a_user

  # value objects make testing easier, testing for what we care about, not every
  # notification
  assert result.notifications.any? {|n| n.message == "New user 'pizza' created"}
end
```

### Avoid dynamically generating tests

Here's an exception to keeping code DRY. Explicit makes it easy to jump to a specific line and be on a specific test.

```ruby
# bad
[ :empty, :normal ].each do |type|
  [ :read, :write ].each do |action|
    test "handles #{action} to #{type} wikis" do
    end
  end
end

# good
def test_handles_read_to_empty_wikis
end

def test_handles_write_to_empty_wikis
end

def test_handles_read_to_normal_wikis
end

def test_handles_write_to_normal_wikis
end
```

### Prefer built in assertions over custom ones

It's tempting to group assertions together into test helper methods. Unless it's a really general helper method, these inevitabily become stale and unused over time.

```ruby
# bad
def test_runs_pre_receive_hooks
  # it's not immediately clear what this helper does
  assert_run_hooks "pre-receive", [/^[0-9a-f]+ [0-9a-f]+ refs\/heads\/master$/]
end

# good
def test_runs_pre_receive_hooks
  assert_includes @hooks_log, "pre-receive"
  assert_match /^[0-9a-f]+ [0-9a-f]+ refs\/heads\/master$/, @refs
end
```

### Minimize setup and fixtures blocks

Create objects needed for test in the test so it's easy to read and reason about

```ruby
# bad
def setup
  @user = User.new
end

def test_user_requires_email
  # have to jump to a different place to know how @user is defined
  refute_predicate @user, :valid?
end

# Good
def test_user_requires_email
  user = User.new
  refute_predicate user, :valid?
end
```

### Use status codes in functional tests

```ruby
# bad
def test_404_when_not_admin
  get "/admin"
  assert_response :not_found
end

# good
def test_404_when_not_admin
  get "/admin"
  assert_response 404
end
```

## Lib

### Avoid depending on Rails

Think of lib as your internal open source repository. Anything within it should
be usable outside of your application.

```ruby
class Bad
  attr_accessor :bar

  # bad assumes rails
  delegate :foo => :bar
end

# good use stdlib
require "forwardable"
class Good
  extend Forwardable

  def_delegator :bar, :foo
end
```
