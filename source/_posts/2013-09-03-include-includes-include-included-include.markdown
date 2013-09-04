---
layout: post
title: "Ruby & Me: include vs other includes"
author: "Alisa Chang"
date: 2013-09-03 23:01
comments: true
categories: ruby rails include module mixins eager-loading
---

{% img left /images/way.jpg 298 235 [Pixel Img] %}
The beautiful thing about ruby is that it wants to be expressive and legible - it aims to mimic human language. Unfortunately human language is not that consistent. Words are not used the same way all the time because usage is often driven by context. The good news is that Ruby makes some allowances for this and provides different ways (aka methods) to help us be more explicit in our programming. 

For example, 'map' is a method that performs the same job as 'collect', and there are times when you might find it preferable to specify that values are being collected from a block vs. when they are being mapped. The truth is though that 'map' has been historically used in functional programming, and might sometimes be used to please the habitual self over semantic purpose.

So there we have it. Po-TAY-toh Po-TAH-toh... but this is America and we say Po-TAY-toh! Seriously though, it's best to use whatever language helps you and others easily read your code. But what happens when terms are the same, or overlap between Ruby and Rails? 

I was recently asked to explain what **include** does in Rails, and the first thing that came to mind was the ruby method include?(obj) - a boolean method that returns true if the given object is present in self:
    grades = ["A", "C+", "B-"]
    grades.include?("A") #=> true
    grades.include?("F") #=> false

But 'include?'' is a ruby method, and the question specified Rails. Hm. Well, I've used 'include' in Ruby to mix in module instance methods, but I simply haven't had to use it in Rails. So later in the day I did what any self-respecting nerd would do, and looked its usage up in RailsGuide. I also asked my classmates and instructor the same question. Sample responses included: like where in Rails? in a mixin? a module? in eager loading? what do you mean?

As Tony the Tiger says: grrreat.

This post briefly explores the various uses of a similarly named method between Ruby (1.9.3) and Rails (3.2.13), using the variations of "include" as an example. The include method is not by any means the most exciting one, but it is one that is commonly used in different contexts.

In Ruby, many methods such as **include?** can be similarly applied to various contexts with slight variations:
    array.include?(true_if_this_object_is_present_in_array)
      [1,2,3,4].include?(2) #=> true
      [1,2,3,4].include?(5) #=> false

    hash.include?(true_if_this_key_is_present_in_hash)
      {:name => "Bob", :age => 25}.include?(:name) #=> true
      {:name => "Bob", :age => 25}.include?(:gender) #=> false

    "string".include?(true_if_this_string_or_character_is_present_in_string)
      "hello".include?("el") #=> true
      "hello".include?("pi") #=> false
    
    (range "this".."that").include?(true_if_this_object_is_an_element_of_range)
      (34..76).include?(38) #=> true
      (34..76).include?(14) #=> false

    environment_variable.include?(true_if_there_is_an_environment_variable_with_this_name)
      PATH = "/Users/LittleApple/.rvm/gems/ruby-1.9.3-p429/bin"
      ENV['PATH'].include?('bin') #=> true
      ENV['PATH'].include?('BigApple') #=> false

    module.include?(true_if_this_module_is_present_in_the_module_or_one_of_the_module's_ancestors)
      module Granny
      end

      class Mama
        include Granny
      end
      
      class Baby < Mama
      end
      
      Mama.include?(Granny)   #=> true
      Baby.include?(Granny)   #=> true
      Granny.include?(Granny) #=> false

There are Public Class methods, Private Class methods, Module methods, Enumerable methods, and so on. This leads to some additional variations on the usage of 'include' in Ruby:
    Module.include(module, ...) 
      #=> used to mix module constants, methods, and variables into the Module or class instances.

    Module.included_modules 
      #=> returns an array list of modules that are included in Module.

    Module.included(other_module)
      Module Hello
        def Hello.included(other_module)
          (...) 
        end
      end

      Module
        include Hello
      end
      #=> callback is invoked when the (receiver) is included in another module or class. 

Moving onto Rails.

ActiveRecord in Rails is an object relational mapping system that lets you retrieve objects from your database using queries that execute as SQL, without actually having to write raw SQL. Includes is one of the many finder methods available in ActiveRecord that helps minimize the number of queries run against your database. 

With the includes method, all associations can be specified in advance by passing arguments. This act of preloading is also known as eager loading, and is how ActiveRecord tackles N+1 - a problem that occurs with associations. When you load the parent object and start loading one child or association at a time, you end up having to run N+1 queries. Pre-loading associations minimizes the number of queries by way of its specificity:
    School.includes("students").where(first_name: 'Ryan', students: {status: 'accepted'}).count

The includes method can come in handy when you want to display the same information in multiple views but don't want to bog down performance by repeatedly running the same query. To provide even more flexibility, Rails has an :include option for those times that a different finder method might be better suited to your query, but you still want to eager load associations. 

Since Rails runs on the Ruby programming language, the various 'include' ruby class and module methods can be used to respectively drive object behavior or refer to modules, mix-ins or concerns. 

So there you have it. The many wonderful uses of the term 'include'.
