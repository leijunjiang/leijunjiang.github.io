---
layout: post
title:  "metaprogramming: class_eval"
date:   2024-06-05 14:00:20 +0200
categories: jekyll update
tags: metaprogramming class_eval
---
[try ruby online here](https://onecompiler.com/ruby/) 

```ruby
class A
  B = "b"
  @@a = 1

  def a 
    @@a
  end
end

```
Question, how can we use class_eval to reassign the value of B and @@a so that

B = 'b2'

@@a = 2

&nbsp;

{::options parse_block_html="true" /}

<details><summary markdown="span">check the answer</summary>

```ruby
A.class_eval "@@a = 2"
A.class_eval "B.replace('b2')"

p A.new.a
# => 2

p A::B
#=> 'b2'
```

</details>
{::options parse_block_html="false" /}

&nbsp;
&nbsp;

```ruby
module B
  def hi
    p 'hi'
  end
end

class C 
  def self.set_logger method_name
  end

  include B

  set_logger :hi
end
```

Question, how can we do that we don't need to write the code?
```
set_logger :hi

in the class C
```

&nbsp;

{::options parse_block_html="true" /}

<details><summary markdown="span">check the answer</summary>
```ruby
module B
  def self.included base
    base.class_eval do
      set_logger :hi
    end
  end

  def hi
    p 'hi'
  end
end

class C
  def self.set_logger mthode_name
  end

  include B
  set_logger :hi
end
```
</details>
{::options parse_block_html="false" /}
