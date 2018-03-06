---
title: "Modules in Modules in Rails"
date: 2018-03-06T13:33:32-05:00
draft: false
---

#### Metaprogramming: Including Modules to Classes using Modules

tl;dr: we can use `klass.class_eval { includes SomeNewModule }` or `klass.send(:include, SomeNewModule)` from a current module to dynamically include a newly desired module into a bunch of classes that use the current module.

#### The Problem

Suppose we have a few different vehicle classes like `Plane` and `Rocket`. `Plane` and `Rocket` (and any other flying vehicles) need to know how to `take_off` and `land` so they include the `CanFly` module.

```ruby
class Plane
  include CanFly
  attr_accessor :speed

  def fly_somewhere
    take_off
    change_speed(100)
    land
  end
end

class Rocket
  include CanFly
  attr_accessor :speed

  def go_to_space
    take_off
    change_speed(1000)
    land
  end
end

module CanFly
  def take_off
    #take off logic
  end

  def land
    #landing logic
  end

  def change_speed(new_speed)
    self.speed = new_speed
  end
end
```

Great! Each of our flying classes have access to the methods they need to fly. But suppose we then create a new class called `Car`. `Car` doesn't need to take off or land. But it does need to change speed. So what do we do? Well, one option is to create a new concern, that we can use across each of our models.

So we just include this new module in my `Car` class and we're good to go. Right?

```ruby
module CanFly
  #...other methods

  def change_speed(new_speed)
    self.speed = new_speed
  end
end

# new module with common code
module CanAccelerate
  def change_speed(new_speed)
    self.speed = new_speed
  end
end

# new vehicle class
class Car
  include CanAccelerate
  attr_accessor :speed

  def drive_somewhere
    change_speed(50)
  end
end
```

Except, now we've repeated the same `change_speed` method in two different places. So that's not great...

We could delete this method from the `CanFly` module, but then we'd have to include the `CanAccelerate` module to each one of our vehicles that use `change_speed` from the `CanFly` module.

More importantly, any time we create a new class that needs `CanFly`, we'd need to remember to also include the `CanAccelerate` module as well. That's awful!

#### The Solution

We could use a little metaprogramming technique here to update all the classes that use `CanFly` to also include `CanAccelerate` dynamically. In other words, this is useful when we want to make a change to a class(es) without explicitly knowing what it is until runtime. To do that, we can use `klass.class_eval` or `klass.send`.  

`klass.class_eval` takes a block in which the context of the block is the klass itself, not the module. So, to include the `CanAccelerate` module for all classes that already include `CanFly`, we can do something like this:

```ruby
module CanFly
  def self.included(klass)
    klass.class_eval do
      includes CanAccelerate # => so this includes CanAccelerate for that klass
    end
  end
end
```

Alternatively, this would also work:
```ruby
module CanFly
  def self.included(klass)
    klass.send(:include, CanAccelerate) # => no context change here
  end
end
```

I prefer the first method stylistically in the event of adding multiple other concerns/modules. It's a little less verbose. But the two seem interchangeable to me in this scenario.

Thanks for reading!
