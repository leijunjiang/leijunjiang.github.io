---
layout: post
title:  "metaprogramming: method_missing define_method"
date:   2024-06-05 12:00:20 +0200
categories: jekyll update
tags: metaprogramming method_missing define_method
---

[try ruby online here](https://onecompiler.com/ruby/) 

In ruby, there many ways to define getter and setter methods.

# simple way

```ruby
class A
  @@attributes = {}

  def self.title
    @@attributes[:title]
  end

  def self.title= value
    @@attributes[:title] = value
  end
end
```

&nbsp;
&nbsp;

# using method_missing

```ruby
class B
  @@attributes = {}

  class << self
    def method_missing method_name, *params
      method_name = method_name.to_s

      if method_name =~ /=$/
        @@attributes[method_name.sub('=', '')] = params.first
      else
        @@attributes[method_name]
      end
    end
  end
end
```

&nbsp;
&nbsp;

# define_method

in ruby, when we use enum, we can code like this
```ruby
user_a.is_pending?
user_a.is_activated?
```

now let's use define_method to implement this.

```ruby
class User < ActiveRecord::Base
  STATUS = %w[pending activated suspended]

  STATUS.each do |status|
    define_method "is_#{status}" do
      self.status == status
    end
  end
end
```

