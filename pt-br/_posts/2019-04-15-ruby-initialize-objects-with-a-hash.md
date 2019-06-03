---
layout: post
title: "Nota Rápida #1 - Inicializar objetos em Ruby passando argumentos nomeados com um Hash"
lang-ref: ruby-initialize-objects-with-a-hash
lang: pt-BR
locale: pt-BR
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

Desde que eu comecei a trabalhar como Engenheiro de Software na Peerdustry,
eu tenho tido muita dificuldade em encontrar tempo para fazer tudo que eu
gostaria fora das minhas horas de trabalho. Consequentemente, eu tenho contribuído
para projetos de software livre com uma frequência bem mais baixa do que eu
gostaria e não consegui nem sequer escrever um artigo para este blog em 2019.
Entretanto, tenho progredido recentemente na organização do meu tempo.
Em relação à escrita, eu vou tentar uma abordagem mais minimalista para
aumentar a frequência de escrita através da produção de textos mais curtos,
com **notas rápidas** ou dicas pequenas que são úteis para mim e para outras
pessoas.

Esse é o meu primeiro artigo onde aplico essa abordagem. Ele contém uma pequena
dica de como inicializar seus objetos de Ruby puro passando argumentos nomeados
com um Hash.

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
