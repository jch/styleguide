# Ruby and Rails styleguide

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
