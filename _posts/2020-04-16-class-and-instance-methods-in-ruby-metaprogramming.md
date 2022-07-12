---
title:  "Class And Instance Methods in Ruby Metaprogramming"
date:   2020-04-16 09:00:00
permalink: class-and-instance-methods-in-ruby-metaprogramming
---

This is my own summary on the class and instance methods in relation to metaprogramming in ruby.

## Motivation

Many articles out there have already given a detailed write up on this topic. Here I am giving my 2 cents, partly to help me revise when I stumble on this concept again. Because this is how I understand it.

The articles written by various other experienced developers are by no means inadequate. I guess it just people having different learning styles and so here I am documenting mine intending to serve only 1 audience, ie me 🙂

## Start With The Syntax

I stumbled on this topic because of the various different syntaxes I have seen in various ruby code bases. And they are nothing like ruby. Weird symbols that go against ruby’s English like syntax and blocks with deep implicit meaning that brings about confusion when reading the code are some examples.

So I thought it is best to come clear with the syntaxes first.

## Instance Methods

So there are a couple of ways to define instance methods in ruby.

```ruby
class Instance1
  def hello
    p "self inside hello is #{self}"
  end

  class_eval do
    def hello
      p "self inside class_eval hello is #{self}"
    end
  end
end

class Instance2
  class_eval do
    def hello
      p "self inside class_eval hello is #{self}"
    end
  end

  def hello
    p "self inside hello is #{self}"
  end
end
```

Here are 2 classes Instance1 and Instance2 with the same method names and definitions. The only difference is the order which they are defined. And if we take a look at their output:

```ruby
instance1 = Instance1.new
instance2 = Instance2.new

instance1.hello # "self inside class_eval hello is #<Instance1:0x00007ff17da771c0>"
instance2.hello # "self inside hello is #<Instance2:0x00007ff17d7bbb98>"
```

The method that is defined later will run. The significance here is that these 2 definitions are in fact the same thing. They are both legit but different ways to define instance methods, and the methods defined later will override the one defined initially as expected.

## Class Methods

There are 3 different ways based on my research.

```ruby
class Class1
  def self.hello
    p "self inside self.hello is #{self}"
  end

  instance_eval do
    def hello
      p "self inside instance_eval hello is #{self}"
    end
  end

  class << self
    def hello
      p "self inside class << self is #{self}"
    end
  end
end

class Class2
  instance_eval do
    def hello
      p "self inside instance_eval hello is #{self}"
    end
  end

  class << self
    def hello
      p "self inside class << self is #{self}"
    end
  end

  def self.hello
    p "self inside self.hello is #{self}"
  end
end

class Class3
  class << self
    def hello
      p "self inside class << self is #{self}"
    end
  end

  def self.hello
    p "self inside self.hello is #{self}"
  end

  instance_eval do
    def hello
      p "self inside instance_eval hello is #{self}"
    end
  end
end
```

The same thing here. 3 classes with the same 3 methods defined in different order. And yes, the output will be dictated by last one that is defined.

```ruby
Class1.hello # "self inside class << self is Class1"
Class2.hello # "self inside self.hello is Class2"
Class3.hello # "self inside instance_eval hello is Class3"
```

No shit. They are just different ways to do the same thing.

## Best Practices

So now with the confusion over different syntaxes out of the way, let’s refer to a single class for the rest of the article.

```ruby
class MyClass
  class_eval do
    def hello
      p "self inside class_eval hello is #{self}"
    end
  end

  instance_eval do
    def hello
      p "self inside instance_eval hello is #{self}"
    end
  end
end

MyClass.hello # "self inside instance_eval hello is MyClass"
MyClass.new.hello # "self inside class_eval hello is #<MyClass:0x00007ff188369ad0>"
```

These are the best practices to define a class method and an instance method in the metaprogramming way. The instance_eval method is especially important, in my humble opinion, in replacing the class << self syntax that is so baneful in ruby linguistics.

INSERT MEME


## Why The Need For a Different Way To Define A Method?

One of the purpose of metaprogramming derives from the need to define methods during runtime. They are often used in conjunction with the method define_method to define new methods based on variables that are only available during run time.

An example would be the current_user method in the authentication related [devise gem](https://github.com/heartcombo/devise). If you have multiple models, like AdminUser and Player on top of User, you can easily access an instance of them in the context of the controller via current_admin_user and current_player respectively. This is done without you having to copy paste the content of current_user into these “new” methods.

This is made possible due to metaprogramming. devise defines these new methods at runtime, looking at all the models that require its involvement, and generate these helper methods all without writing extra code.

The robustness of metaprogramming is clearly crucial and essential.

## The Essence of Instance and Class of a Class

So there is this whole confusion about a hidden class in a class in ruby. This all stems from 1 fact: everything in ruby is an object. That includes a class.

INSERT MEME Everything is an object in ruby

So how can a class as we know it, with all its inheritance and class method and instance method properties, be an instance of a ruby object?

This is made possible with the existence of a hidden metaclass whenever a class is defined in ruby. This hidden class holds the common properties of classes as we know them and allow us to use mere ruby objects like a class. There’s a lot of confusion in this sentence due to the overlapping usage of the word class. Be sure to read it again.

Hence class methods are in fact instance methods of this metaclass. These methods of the metaclass are not inherited by the instances of the class, just like how a class method should be.

## Singleton Class In Ruby

Another name for these hidden metaclasses is singleton class (‘eigenclass’ is another). I find this naming more apt in the context of ruby (And I will refer to it as singleton class from here onwards). Allow me to explain.

Classes should have a unique namespace in its codebase. Therefore when a class is defined, and so for its corresponding metaclass, it will not be defined again. Its one and only instance of itself will thus exist for the lifetime of the application with no duplicate. This gives the term ‘singleton’ so much more sense.

In fact, I perceive it as the official definition in ruby because of the method singleton_class which gives an object instance access to its “metaclass” instance. Here is an example.

```ruby
MyClass.new.singleton_class.hello # "self inside instance_eval hello is #<Class:#<MyClass:0x00007faa7a3bb448>>"
```

Note the memory address of the singleton class. It indicates that this singleton class is in fact an instance.

## instance_eval and class_eval

With a better concept of a class, a singleton class and an instance, let’s look at instance_eval and class_eval block.

Initially, it came across to me as unintuitive that a class method is defined under instance_eval, and an instance method is defined under class_eval. Why make life so difficult?

However, once we understand the concept, everything will fall into place.

Under `instance_eval`, we are looking at the MyClass under the context that it is an instance. We are evaluating it as an instance. And an instance of a class can only refer to the singleton class that will hold anything that should possess the properties of a typical class that we know in computing.

Under `class_eval`, we are evaluating MyClass as a class, where we define methods to be applied on instances of MyClass as usual.

These 2 methods determine the context in which the methods in it are defined. In particular, it dictates what the variable self refers to in each scope. This [article](https://yehudakatz.com/2009/11/15/metaprogramming-in-ruby-its-all-about-the-self/) has a much detailed explanation on that.

## Conclusion

There’s definitely more to metaprogramming that to define methods during runtime. This idea of using code to generate code has immense potential and this may just be only the tip of the iceberg.
