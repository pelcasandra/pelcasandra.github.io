---
permalink: model-composition-in-rails
layout: default
title:  "Model Composition in Rails"
date:   2020-12-13 02:56:33 -0300
categories: rails
language: ruby
---

I recently came with the need of treating two different models alike so as to be able to query them as one ActiveRelation and also trigger their methods using a unified interface even if they are internally differently from each other.

Let's say we have `Business` and `Person` models, each of them has their own database table and I want a way to retrieve all businesses and people in one single query using *ActiveRecord* and do it consistently returning an *ActiveRelation* object.

We can reach this by introducing a new model namely `Entity` that will take either a business or a person as an *entitable* polymorphic association. 

## Multi-table inheritance with delegated type

I was lucky to find that Rails 6.1 introduced the concept of [delegated types](https://edgeapi.rubyonrails.org/classes/ActiveRecord/DelegatedType.html) the same day I needed to do this. 

```ruby
class Entity
  delegated_type :entitable, types: %w[ Business People ]
end
```

And on the `Business` and `People` models you add:

```ruby
class Business
  included do
    has_one :entity, as: :entitable, touch: true
  end
end
```

Now you can do the following:

```ruby
# Create a business
Entity.create entitable: Business.create(name: 'Acme' )

# Create a person
Entity.create entitable: Person.create(name: 'Martin' )
```

And of course you can now retrieve both businesses and people together with `Entity.all`. You can even create scopes for doing some filtering.

```ruby
Entity.all.businesses
Entity.all.people
```

This is a nice addiction to Rails. It looks great and it helps maintain the code simple and easier to read.

## Delegating methods gets messy quickly

Now, let's assume that every person has an associated `Identity` model that returns their national identification number. We can easily delegate the method `number` from  `Person` to the `Identity` association:

```ruby
class Person 
  belongs_to :identity

  delegate :number, to: :identity, prefix: true
end
```

We can call `person.identity_number` to retrieve the identity number directly from the person object without passing through the identity object, honoring the [Law of Demeter](https://en.wikipedia.org/wiki/Law_of_Demeter). 

The `prefix: true` prefixes the method as `identity_number` instead of `number` as it is easier to understand that we're calling the identity number: we want to avoid  `person.number` because people don't have numbers, they have identification numbers.

Now, let's assume a business may belong to a person.

```ruby
class Business
  belongs_to :person
end
```

If we know that entities always have an identity either directly from a person or through a business's owner we want to be able to call the identity from the entity with the same method regardless of the type of type of *entitable* object. 

From an entity we would want to be able do something like:

```ruby
entity.person_identity_number
=> "4WA3X6E21T"
```

To do this, we can attempt to add delegate method in the `Entity` model so `Person` and `Business` will receive the delegated method `person_identity_number` without its prefix as `identity_number`. We can try now to re-delegate from the person to the identity and from the business to the person. 

```ruby
class Entity
  # ...

  delegate :identity_number, to: :entitable, prefix: :person
end

class Person
  # ...

  # Wrong! We already defined :number above.
  delegate :identity_number, to: :identity 
end

class Business
  # ...

  delegate :identity_number, to: :person
end
```

However we quickly run into a problem: How to delegate this method down the line? 

We are passing a method called `identity_number` to `Person` when it is already delegating the method as `number` with prefix `identity` (the prefix is removed when on the Identity model so the method simply becomes `identity.number`.

## A compositionable solution!

After spending some time over this conundrum I came to the conclusion that a possible solution may look like this:

```ruby
class Entity
  delegated_type :entitable, types: %w[ Business People ]

  delegate :identity, to: :entitable, private: true
  delegate :number, to: :identity, prefix: :person_identity
end

class Person
  belongs_to :identity
end

class Business
  belongs_to :person

  delegate :identity, to: :person
end
```

Now you can obtain an identity number regardless of where it is coming from a person or from a person who a company belongs to.

```ruby
<%= entity.person_identity_number %>
```

Behind the scenes we delegated the association `identity` from `Entity` to `:entitable` (Business or Person). If they do not handle it directly, they can pass delegate it again (as the `Business` does delegate it to `Person`). 

You can now implement multiple models that can [cuack](https://en.wikipedia.org/wiki/Duck_test) as entitables if they either implement or delegate the `identity` method.

Now, it may or it may not make sense to use delegates. I guess it depends on the complexity of the models. On one side delegate make it simple to understand the methods that are delegated on the other side it may be simpler to simple define a method to perform the delegation. This is also equivalent:

```ruby
class Entity
  delegated_type :entitable, types: %w[ Business People ]

  def person_identity_number
    entitable.identity_number
  end
end

class Person
  belongs_to :identity

  def identity_number
    identity.number
  end
end

class Business
  def identity_number 
    person.identity_number
  end
end
```

I found it interesting to explore object composition using delegate and multi-table-inheritance as this is an interesting pattern to help keep the code simpler and easier to read. 
