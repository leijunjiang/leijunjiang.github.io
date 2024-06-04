---
layout: post
title:  "metaprogramming: singleton methods"
date:   2024-06-04 14:31:20 +0200
categories: jekyll update
tags: metaprogramming class singleton methods self
---

## Singleton Methods

[try ruby online here](https://onecompiler.com/ruby/) 
```ruby
#1
class A
  def self.hi
    p 'hi'
  end
end

#2
class A
  class << self
    def hello
      p 'hello'
    end
  end
end

#3
def A.hey
  p 'hey'
end

#4
(class << A; self; end).class_eval do
  def you
    p 'you'
  end
end

#5
A.singleton_class.class_eval do
  def folk
    p 'folk'
  end
end

A.hi
A.hello
A.hey
A.you
A.folk
```

Output:
```
"hi"
"hello"
"hey"
"you"
"folk"
```

&nbsp;
&nbsp;

## self
```ruby
class A
  p "first self = #{self}" 

  class << self
    p "second self = #{self}" 
  end

  def hello
    p "third self = #{self}" 
  end
end

p A
A.new.hello
```

{::options parse_block_html="true" /}

<details><summary markdown="span">check the output:</summary>

```ruby
p A
# ==> A

A.new.hello

# ==> "first self = A"
# ==> "second self = #<Class:A>"
# ==> "third self = #<A:0x0000557d663548d0>" 
```

</details>

{::options parse_block_html="false" /}


&nbsp;
&nbsp;

## Variables

```ruby
class B
  @a = 1
  @@b = 2

  def initialize
    @c = 3
    @@d = 4
  end

  class << self
    @e = 5
    @@f = 6
  end

  def a 
    @a
  end

  def c
    @c
  end
end

b = B.new
p B.instance_variables
p B.class_variables
p b.instance_variables
p B.singleton_class.instance_variables
p B.singleton_class.class_variables
```

{::options parse_block_html="true" /}

<details><summary markdown="span">check the output</summary>

```ruby
b = B.new
p B.instance_variables
# ==> [:@a]

p B.class_variables
# ==> [:@@b, :@@f, :@@d]

p b.instance_variables
# ==> [:@c]

p B.singleton_class.instance_variables
# ==> [:@e]

p B.singleton_class.class_variables
# ==> [:@@b, :@@f, :@@d]
```

</details>

{::options parse_block_html="false" /}


