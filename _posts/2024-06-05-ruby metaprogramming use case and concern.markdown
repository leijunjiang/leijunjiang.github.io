---
layout: post
title:  "metaprogramming: use case and concern"
date:   2024-06-05 16:00:20 +0200
categories: jekyll update
tags: metaprogramming use case concern
---
[try ruby online here](https://onecompiler.com/ruby/) 

Suppose we have a class like this
```ruby
class Device
  def latest_data
    {
      "data" => {
        "node_info" => "this is a sensor",
        "battery" => 90
      },
      "device_type" => "Sensor"
    }
  end
end
```

and we want to call the data by method
```ruby
d = Device
p d.node_info
p d.battery
```

with metaprogramming we can write elegant methods like this
```ruby
class Device
  include ActsAsField

  field :device_type, "device_type"
  field :battery, "data.battery"
  field :node_info, "data.node_info"

  field :battery_to_text, proc { |device| "#{device.battery}%" }

  def latest_data
   {
      "data" => {
        "node_info" => "this is a sensor",
        "battery" => 90
      },
      "device_type" => "Sensor"
    }
  end
end
```

Now lets see what this module ActsAsField looks like

&nbsp;

{::options parse_block_html="true" /}

<details><summary markdown="span">check the answer</summary>

```ruby
module ActsAsField
  def self.included base
    base.include InstanceMethods
    base.extend ClassMethods

    base.class_eval do
      @@acts_as_fields = []
    end
  end

  module ClassMethods
    def field name, path
      result = class_variable_get(:@@acts_as_fields)
      result << name.to_sym
      class_variable_set(:@@acts_as_fields, result)
    

      define_method(name) do
        case path
        when String
          path.split(".").inject(self.latest_data) { |data, key| data[key] }
        when Proc
          path.call(self)
        end
      end
    end
  end

  module InstanceMethods
    def acts_as_field

    end
  end
end
```

</details>

{::options parse_block_html="false" /}

&nbsp;
&nbsp;

## concern
Let's look at this module ActsAsField

```ruby
module ActsAsField

  def self.included base
    base.include InstanceMethods
    base.extend ClassMethods

    base.class_eval do
      @@acts_as_fields = []
    end
  end

  module ClassMethods
    def class_method
    end
    # ...
  end

  module InstanceMethods
    def instance_method
    end
    # ...
  end
end

```

If we use **active_support/concern**, we can have code like this
```ruby
require 'active_support/concern'

module ActsAsField
  extend ActiveSupport::Concern

  included do
    @@acts_as_fields = []
  end

  class_methods do
    def class_method
    end
  end

  def instance_method
  end
end

```
