Wrapping Attributes
===================

With the method [wrap] you can take some attributes from a tuple and wrap them to either a nested tuple, or a model.

[wrap]: http://www.rubydoc.info/gems/rom/ROM/Mapper/AttributeDSL:wrap

* [Basic Usage](#basic-usage)
* [Removing Prefixes](#removing-prefixes)
* [Renaming Attributes](#renaming-attributes)
* [Wrapping to Model](#wrapping-to-model)
* [Applying another Mapper](#applying-another-mapper)
* [Nesting Wrappers](#nesting-wrappers)
* [Edge Cases](#edge-cases)

See [Unwrapping Tuples](unwrapping.md) for the inverse transformation of data.

Notice, mappers have [high-level and low-level API](index.md#high-level-and-low-level-api). Examples in this section use the high-level API only. The same syntax is applicable to low-level API as well.

Basic Usage
-----------

Suppose there is a predefined relation that returns an array of tuples:

```ruby
users = ROM.env.relations(:users)
users.first
# { id: 1, name: "Joe", contact_email: "joe@example.com", contact_skype: "joe" }
```

With a `wrap` we can take `:contact_email` and `:contact_skype` attributes and convert them into the `:contact` tuple:

### Inline Syntax

```ruby
class WrappedUsersMapper < ROM::Mapper
  register_as :wrapped_users
  relation :users

  wrap contact: [:contact_email, :contact_skype]
end

users.as(:wrapped_users).first
# {
#   id: 1, name: "Joe",
#   contact: { contact_email: "joe@example.com", contact_skype: "joe" }
# }
```

### Block Syntax

Just the same result can be obtained by using the block with attributes defined inside:

```ruby
class WrappedUsersMapper < ROM::Mapper
  register_as :wrapped_users
  relation :users

  wrap :contact do
    attribute :contact_email
    attribute :contact_skype
  end
end

users.as(:wrapped_users).first
# {
#   id: 1, name: "Joe",
#   contact: { contact_email: "joe@example.com", contact_skype: "joe" }
# }
```

Removing Prefixes
-----------------

Use `:prefix` to remove prefixes from wrapped attributes. The `attribute` method arguments should have no prefixes:

### Inline Syntax

```ruby
class WrappedUsersMapper < ROM::Mapper
  register_as :wrapped_users
  relation :users

  wrap :contacts, [:email, :skype], prefix: 'contact'
end

users.as(:wrapped_users).first
# {
#   id: 1, name: "Joe",
#   contacts: { email: "joe@example.com", skype: "joe" }
# }
```

Use `:prefix_separator` in case of a custom separator:

```ruby
user.first
# {
#   id: 1, name: "Joe",
#   :"contact.email" => "joe@example.com", :"contact.skype" => "joe"
# }

class WrappedUsersMapper < ROM::Mapper
  register_as :wrapped_users
  relation :users

  wrap :contacts, [:email, :skype], prefix: 'contact', prefix_separator: '.'
end

users.as(:wrapped_users).first
# {
#   id: 1, name: "Joe",
#   contacts: { email: "joe@example.com", skype: "joe" }
# }
```

### Block Syntax

Alternatively, use the [prefix](prefix.md) method inside the block.

The method should have no attributes and cannot customize either prefix, or its separator. It takes the prefix from the name of the `wrap` and separates it by the underscore `"_"`:

```ruby
class WrappedUsersMapper < ROM::Mapper
  register_as :wrapped_users
  relation :users

  wrap :contact do
    prefix
    attribute :email
    attribute :skype
  end
end

users.as(:wrapped_users).first
# {
#   id: 1, name: "Joe",
#   contact: { email: "joe@example.com", skype: "joe" }
# }
```

Renaming Attributes
-------------------

Inside the block use the `:from` option of the [attribute](attribute.md) method:

```ruby
class WrappedUsersMapper < ROM::Mapper
  register_as :wrapped_users
  relation :users

  wrap :contact do
    attribute :to_write, from: :contact_email
    attribute :to_chat,  from: :contact_skype
  end
end
```

**Notice** this feature requires the block syntax. It cannot be done inline.

### Edge Cases

The method works just fine when the name of wrapped tuple is the same as one of its attributes. There is no need for renaming attributes.

```ruby
meetings = ROM.env.relation(:meetings)
meetings.first
# { place: "The Conference Hall", agenda: "Future plans", main_thesis: "Bankruptcy" }

class MeetingsMapper < ROM::Mapper
  register_as :wrapped_meetings
  relation :meetings

  wrap :agenda, [:agenda, :thesis]
end

meetings.as(:wrapped_meetings).first
# { place: "The Conference Hall", agenda: { agenda: "Future plans", main_thesis: "Bancrupcy" } }
```

Wrapping to Model
-----------------

Define the [model](model.md) to map wrapped tuple into:

```ruby
require "ostruct"

class UserMapper < ROM::Mapper
  register_as :entity
  relation :users

  wrap :contact do
    model     OpenStruct
    attribute :contact_email
    attribute :contact_skype
  end
end

users.as(:entity).first
# {
#   id: 1, name: "Joe",
#   contact: <OpenStruct contact_email="joe@example.com", contact_skype="joe">
# }
```

**Notice** this feature requires the block syntax. It cannot be done inline.

Applying another Mapper
-----------------------

Another mapper can be applied to wrapped group of attributes. To do this, use the `:mapper` inline option:

```ruby
class ContactMapper < ROM::Mapper
  register_as :contact
  relation :users

  attribute :email, from: :contact_email
  attribute :skype, from: :contact_skype
end

class UserMapper < ROM::Mapper
  register_as :hash
  relation :users

  wrap :contacts, mapper: ContactMapper
end

users.as(:entity).first
# { id: 1, name: "Joe", contacts: { email: "joe@doe.org", skype: "joe" } }
```

### Edge Cases

Don't define attributes inline along with the `:mapper` option! In this case the mapper won't be applied:

```ruby
class ContactMapper < ROM::Mapper
  register_as :contact
  relation :users

  attribute :email, from: :contact_email
  attribute :skype, from: :contact_skype
end

class UserMapper < ROM::Mapper
  register_as :hash
  relation :users

  wrap contacts: [:name], mapper: ContactMapper
end

users.as(:entity).first
# { id: 1, contacts: { name: "Joe" }, contact_email: "joe@doe.org", contact_skype: "joe" }
```

Don't use block along with the `:mapper` option! All definitions that are made inside the block will be ignored:

```ruby
class ContactMapper < ROM::Mapper
  register_as :contact
  relation :users

  attribute :email, from: :contact_email
  attribute :skype, from: :contact_skype
end

class UserMapper < ROM::Mapper
  register_as :hash
  relation :users

  wrap :contacts, mapper: ContactMapper do
    attribute :name
  end
end

users.as(:entity).first
# { id: 1, name: "Joe", contacts: { email: "joe@doe.org", skype: "joe" } }
```

Nesting Wrappers
----------------

Wrappers can be nested at many levels. You can define a corresponding model for any level of nesting:

```ruby
class UserMapper < ROM::Mapper
  register_as :entity
  relation :users

  wrap :contacts do
    model Contacts

    wrap :email do
      model Messages
      attribute :address, from: :contact_email
    end

    wrap :skype do
      model Skype
      attribute :user, from: :contact_skype
    end
  end
end

users.as(:entity).first
# {
#   id: 1, name: "Joe",
#   contacts: <Contacts
#     @email=<Email @address="joe@example.com">,
#     @skype=<Skype @user="joe">
#   >
# }
```

Edge Cases
----------

### Rejecting Keys

The method `wrap` provides its output regardless of the `reject_keys` setting:

```ruby
class WrappedUsersMapper < ROM::Mapper
  register_as :wrapped
  relation :users
  reject_keys false # is set by default

  wrap :contacts, [:email, :skype], prefix: 'contact'
end

users.as(:wrapped_users).first
# {
#   id: 1, name: "Joe",
#   contacts: { email: "joe@example.com", skype: "joe" }
# }
```

When a `reject_keys` is set to `true`, the `wrap` method defines attributes by itself:

```ruby
class WrappedUsersMapper < ROM::Mapper
  register_as :wrapped
  relation :users
  reject_keys true

  wrap :contacts, [:email, :skype], prefix: 'contact'
end

users.as(:wrapped_users).first
# { contacts: { email: "joe@example.com", skype: "joe" } }
```

Notice the method *always* removes attributes from the upper-level tuple - even in case they are declared explicitly:

```ruby
class WrappedUsersMapper < ROM::Mapper
  register_as :wrapped
  relation :users
  reject_keys true

  attribute :contact_email
  attribute :contact_skype

  wrap :contacts, [:email, :skype], prefix: 'contact'
end

users.as(:wrapped_users).first
# { contacts: { email: "joe@example.com", skype: "joe" } }
```