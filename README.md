# ⚔️ Challenge Answers

## ❓ Questions

- Can you find out where the error happened?
- Can you explain why the code failed now, but not before?
- What would be the options to fix it?
- Do you have any other thoughts on this code?

##  📃 Answers

### 1️⃣ Can you find out where the error happened? Can you explain why the code failed now, but not before?

→ The error happened inside the method `next_display_name` of the class `User::DisplayNameBuilder`. By using the first name and the first letter of the last name, the code generates a display name for the user. Whenever it finds a user with a similar display name, it adds a number to the display name and increases it until it finds an available name. The problem is that many similar users have the same display_name `klaus_m` + number. Because of that, the code tries to increase the last number until 999, and when it reaches that number, it raises the error `StandardError: cannot generate a display_name for this user`

```ruby
@counter >= 999 and raise 'cannot generate a display_name for this user'
```

### 2️⃣ What would be the options to fix it?

→ One of the options to fix this problem would be to change the instance counter variable to

```ruby
@counter = User.where("display_name LIKE ?", "#{@base_display_name}%").count
```

→ As a result, we are now able to determine how many `display_name`s share the same `base_display_name`. Next, we can change the `next_display_name` method to:

```ruby
class User::DisplayNameBuilder
  # [...]

  def next_display_name
    display_name = base_display_name.first(17)
    display_name = "#{display_name}#{@counter}" unless @counter.zero? # When the counter is no longer zero, the counter number is added to the end of the display_name variable.
    return display_name if display_name_available?(display_name)
    
    generate_token(10)
  end

  # [...]
end
```

→ When `display_name` is somehow unavailable, we return the `generated_token` to guarantee that at least it will give the user a display name. As the user is unable to control this generation, **raising an error in this case would only frustrate the user.** 😓

→ Due to those changes, we should also change the tests and add some new ones.

### 💭 Alternatives?

→ Another option to solve this problem is to **increase the maximum counter number**, but I am certain that this **will not scale well** since it will eventually reach the max counter limit for some `display_name` and raise the error again.

→ We could also use the user id instead of the counter variable, but this would put at risk existing `display_name`s that might conflict and **would require extra work for the moment.** In the case of an auto-incremented integer ID, it would also reveal the number of users we have, which may not be desirable.


### 3️⃣ Thoughts on the code

→ There are some things that we can improve.

3.1. On `assign_display_name!`, we can move the early return condition to the callback call in the model, avoiding initializing the class.

```ruby
# app/models/user.rb

class User < ActiveRecord::Base
  # [...]
  before_create { DisplayNameBuilder.new(self).assign_display_name! unless display_name.present? }
  # [...]
end

# app/models/user/display_name_builder.rb

class User::DisplayNameBuilder
# [...]
  def assign_display_name!
    # @user.display_name.present? and return
    @user.display_name = next_display_name
  end
# [...]
end
```

3.2. On the `generate_base_display_name` method, if the full name is blank we can do an early return with the token, as expected.

```ruby
# app/models/user/display_name_builder.rb

class User::DisplayNameBuilder
# [...]
  def generate_base_display_name(user)
    return generate_token(10) if full_name_blank?(user)

    base = escape(user.first_name.to_s[/\A\s*(\S+)/, 1])
    first_last_name_character(user).full? { |char| base << "_#{char}" }
    # PS: I couldn't understand where .full? comes from and what it exactly does.

    base.full?(:first, 20) || generate_token(10)
  end

  def full_name_blank?(user)
    user.first_name.blank? && user.last_name.blank?
  end
# [...]
end
```

3.3. It is possible to make some methods private, based on what we really need to expose from this class.

3.4. EXTRA: we could also add `validate_uniqueness_of :display_name` to the user model if that is what we wish to achieve. In order to accomplish this, we would need to change the callback call to `before_validation :build_display_name, on: :create`.

3.5. EXTRA: the method `generate_token` is not used outside the class, according to the shown files. This indicates that if it is not used anywhere else, it should be refactored, as the default size variable and the block variable are unused.

## ✨ The final code

→ As a result of these changes, the final code would look as follows:

```ruby
# app/models/user.rb

class User < ActiveRecord::Base
  # [...]
  before_create { DisplayNameBuilder.new(self).assign_display_name! unless display_name.present? }
  # [...]
end

# app/models/user/display_name_builder.rb

class User::DisplayNameBuilder
  attr_accessor :base_display_name

  def initialize(user)
    @user = user
    @base_display_name = generate_base_display_name(user)
    @counter = User.where("display_name LIKE ?", "#{@base_display_name}%").count
  end

  def assign_display_name!
    @user.display_name = next_display_name
  end

  private 
  
  def generate_base_display_name(user)
    return generate_token(10) if full_name_blank?(user)

    base = escape(user.first_name.to_s[/\A\s*(\S+)/, 1]) # Ignore leading spaces and fetch the first non-ws token
    first_last_name_character(user).full? { |char| base << "_#{char}" }

    base.full?(:first, 20) || generate_token(10)
  end

  def full_name_blank?(user)
    user.first_name.blank? && user.last_name.blank?
  end

  def escape(str)
    s = I18n.transliterate str.to_s
    s.gsub!(/[^a-z]+/i, '') # remove all non-word characters after transliteration
    s.downcase!
    s
  end

  def first_last_name_character(user)
    escape(user.last_name).first(1)
  end
  
  def next_display_name
    display_name = base_display_name.first(17)
    display_name = "#{display_name}#{@counter}" unless @counter.zero?
    return display_name if display_name_available?(display_name)
    
    generate_token(10)
  end
  
  def display_name_available?(display_name)
    !User.exists?(display_name: display_name)
  end

  def generate_token(size = 64, &block)
    block ||= ->* { true }
    loop do
      token = Tins::Token.new(
        length: size,
        alphabet: Tins::Token::BASE64_URL_FILENAME_SAFE_ALPHABET,
      )
      return token if block.call(token)
    end
  end
end
```

## 🌻 Thank you

Thank you for reading and evaluating my test. It is greatly appreciated. 👨‍💻

All the best,
Gabriel Ferro
