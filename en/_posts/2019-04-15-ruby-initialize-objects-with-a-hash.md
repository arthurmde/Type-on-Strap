---
#
# @!attribute fee
#   @return [Float]
#   The quotation fee to be included in the final price
#
#   == Validations:
#   - presence if status is one of the following: negotiation, accepted, or declined
#   - greater_than_or_equal_to 0.0
layout: post
title: "Quick Note #1 - Initialize Ruby objects passing named arguments with a Hash"
lang-ref: ruby-initialize-objects-with-a-hash
lang: en
locale: en
#feature-img: "assets/img/pexels/taipei-taiwan.jpeg"
#thumbnail: "assets/img/thumbnails/taiwan-wall.png"
categories:
  - Hacking
  - Quick Note
tags:
  - ruby
  - poro
  - dry
  - quick tip
excerpt_separator: <!--more-->
---

Since I started working as a full-time Software Engineer at Peerdustry
I've been having a lot of trouble finding the time to do everything I wanted
outside of my work hours. Consequently, I have contributed to FLOSS projects
less often than I would like and I have not been able to write a single blog post
this year. 
However, I have had some progress recently in organizing my time.
Regarding the writing habit, I will try a more minimalist approach to
improve my frequency by writing short texts with some quick notes or tips
that are useful to others and myself in the future. 

This is my first post where I apply this approach. It has a little tip
about how to initialize your Ruby objects passing named arguments with a Hash.

<!--more-->

<hr>

In a **PORO** - Plain Old Ruby Object, the usual way to assign values to the
attributes of a new object is through the arguments of the *initialize* method,
as illustrated by the snippet below:

```ruby
class Person
  def initialize(name, email, address, phone)
    @name = name
    @email = email
    @address = address
    @phone = phone
  end
end

person = Person.new("Arthur", "arthur@example.com", nil, "+55 11 999999999")
```

The *initialize* method follows the same rules of any other method in Ruby, 
for instance, the parameters must be passed in exactly
the same order in which they were declared. Unlike, the Ruby on Rails framework
(RoR), more specifically the [ActiveRecord](https://guides.rubyonrails.org/active_record_basics.html),
provides a very convenient way to assign values to the object's attributes of your
model classes with a Hash, as shown below:

```ruby
person = Person.new(name: "Arthur", phone: "+55 11 999999999", email: "arthur@example.com")
```

With this approach, we need neither to pass *nil* values nor to follow any
pre-defined order of arguments, since the arguments based on a Hash enable
'named arguments'. Such behavior is also provided by the module
[ActiveModel::Model](https://api.rubyonrails.org/v5.1.6/classes/ActiveModel/Model.html),
which is very useful for including in your Rails App's specific domain objects,
such as [Service Objects](https://medium.com/selleo/essential-rubyonrails-patterns-part-1-service-objects-1af9f9573ca1),
since this module is packaged within the framework.
Besides the support for initializing objects with a hash, the **ActiveModel::Model**
module adds other behaviors to your classes that you may not be interested in,
such as model name introspections, conversions, translations and validations.

However, if you are not using Rails, but still want to initialize your
objects with Hashes, you can create a generic module with
the single purpose of implementing such behavior and include it into your
classes.

```ruby
module HashBasedInit
  def initialize(args)
    args.each do |key, value|
      send("#{key}=", value)
    end
  end
end

class Person
  include HashBasedInit

  attr_accessor :name, :email, :address, :phone
end

person = Person.new(name: "Arthur", phone: "+55 11 999999999", email: "arthur@example.com")
```

Note that it is necessary to declare accessing methods to your attributes as we
did in the **Person** class through the `attr_accessor` method.
Now, we only need to include the **HashBasedInit** module in your other classes
of your project and that's it.

<hr>

<span>**Let's get moving on! ;)**</span>
