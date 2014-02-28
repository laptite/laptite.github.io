---
layout: post
title: "Deep nested attributes using has_many through join model"
author: "Alisa Chang"
date: 2014-02-26 21:34:22 -0500
comments: true
categories: rails deep nesting has_and_belongs_to_many join_association bi-directional_association has_many:through
---

(rails 3.2.15)

It's been almost six months since I graduated from the Flatiron School. It's hard to believe that I haven't written a blog in just as long. Every time I overcame a challenge I kept telling myself that I should write a blog, but never got around to it. 

But this. This problem was a tough nut to crack. I found hints of what to do on StackOverflow and countless articles, but three sources (appended at the end) were most useful. So, definitely check those out.

**SCENARIO:** My company had a very long and convoluted controller. The desire was to simplify it in anticipation for a version update. My task was to wipe the controller clean, redesign the corresponding section, and aim for simple CRUD. But I couldn't get nested_attributes to work. So I broke the hash into pieces and wrote controller methods to get my CRUD actions to work in time for the release. It was shorter and cleaner than the previous version, and handled all validations beautifully. But it just seemed wrong to end up with a fat controller that doesn't take advantage of the magic that is Rails (wink). I've sanitized the code a bit to keep it generic, but it works just the same.

**Problems this blog addresses:**
    - Two-level deep nesting
    - Parent-Child validation (Bi-Directional Association)
    - Assigning multiple drop-down selections to nested object (Join Model Association)
    - Multiple drop-downs from same collection: the last drop-down selection overwrites the first
    - Join table with has_and_belongs_to_many association not working with nested_attributes
    - Nested attributes works only on create action and not on update


**VERSION 1: THE INITIAL CODE THAT DOESN'T AUTOMAGICALLY WORK**

I setup my form using fields_for and accepts_nested_attributes_for following the Rails documentation. Here's a high-level breakdown: 
    
    Car > Make > Pricing
               > Feature ( Select: #{feature_type_id: 1 } ) <-- Color
               > Feature ( Select: #{feature_type_id: 2)} ) <-- Body Type (2-door, 4-door...)

Here's how controller and associations are originally setup in the models, and what the resulting hash looks like:

    class CarsController < ApplicationController
      def new
        @car = Car.new
        make = @car.makes.build
        make.pricings.build
      end

      def create
        @car = Car.new(params[:car])
        if @car.save
          redirect_to edit_car_path(@car), notice: "Car created successfully"
        else
          redirect_to new_car_path(@car), notice: "Car could not be created"
        end
      end
    end

    Models
    class Car < ActiveRecord::Base
      attr_accessible :name, :model_xx, :makes_attributes
      has_many :makes
      has_many :pricings,      through: :makes
      has_many :features,      through: :makes
      has_many :feature_types, through: :features
      accepts_nested_attributes_for :makes, allow_destroy: :true

    class Make < ActiveRecord::Base
      attr_accessible :car_id, :vin, :pricings_attributes, :features_attributes
      belongs_to :car,           touch: true
      has_many   :pricings,      dependent: :destroy
      has_many   :feature_types, through: :features
      has_and_belongs_to_many :features, join_table: :features_makes
      accepts_nested_attributes_for :makes, allow_destroy: true

    class Pricing < ActiveRecord::Base
      attr_accessible :make_id, :price, :currency
      belongs_to :make, touch: true
      has_and_belongs_to_many :makes, join_table: :pricings_makes
      validates  :make_id, :currency, :price, presence: true

    class FeatureType < ActiveRecord::Base
      attr_accessible :name
      has_many  :features, dependent: :destroy
      validates :name,    presence: true
    end

    class Feature < ActiveRecord::Base
      attr_accessible :name, :feature_type_id
      belongs_to :feature_type
      has_and_belongs_to_many :makes, join_table: :features_makes
      validates :name, presence: true
    end

    form.haml
    = form_for @car do |f|
      = f.label :car_name
      = f.text_field :name

      = f.label :model_code
      = f.text_field :model_xx

      = f.fields_for :makes do |v|
        = v.label :vin
        = v.text_field :vin

        = v.fields_for :features do |o|
          = o.label :id, "Color"
          = o.collection_select :id, Feature.where(feature_type_id: 1).order(:name), :id, :name, { prompt: "Select Color" }

          = o.label :id, "Body Type"
          = o.collection_select :id, Feature.where(feature_type_id: 2).order(:name), :id, :name, { prompt: "Select BodyType" }

        = v.fields_for :pricings do |p|
          = p.label :price
          = p.select :currency, Pricing.all.collect{ |p| [currency_symbol(p.currency), p.currency] }, { include_blank: "Currency" }, { class: "selectpicker", data: {style: "btn-primary currency-dropdown"} }
          = p.text_field :price

    params:
    "car"=>{"name"=>"delano", "model_code"=>"dx",
      "makes_attributes"=>{"0"=>{"vin"=>"vin98765",
        "pricings_attributes"=>{"0"=>{"currency"=>"usd", "price"=>"100000"}},
        "features_attributes"=> {"0"=>{"id"=>"6"}, "1"=>{"id"=>"4"}}
      }}
    }

In a perfect world, clicking submit triggers Car.create(params[:car]) and the car auto-magically creates not only itself, but also all of its makes and associations in one fell swoop.

In reality, the car cannot be created for a number of reasons: 
    - Car and Make cannot be saved simultaneously because:
      - Car is not valid because there's no make_id
      - Make is not valid because there's no car_id
      - Pricing is not valid because there's no make_id
    - Make doesn't know what to do with the features hash
    - Last feature selection over-writes the first selection

**BI-DIRECTIONAL ASSOCIATION: Parent-Child Validation for Nested Attributes**

What's causing this error:

    ActiveRecord::RecordNotFound: Couldn't find [table] with ID=1 for ... with ID=

By default, Active Record (AR) creates separate in-memory copies of object data. You need to specify to Rails that there's an inverse relationship between models to optimize object loading. If you don't, AR creates objects with no association to each other. By specifying an inverse relationship, you're telling AR that this Pricing belongs to this make, which belongs to this car. In addition to telling AR how to load the objects, you also need to set the :autosave feature to tell it that you want to save the members simultaneously. By default this is set to false. 

You should also set validations down the line to require each object and specify any dependencies that need to be destroyed along with the parent (unless you want orphaned objects littering your database). Finally, you should add a variant hidden field if you forgot to include it earlier. This facilitates bi-directional relationships through the variant. 

    class Car < ActiveRecord::Base
      ...
      has_many  :makes, inverse_of: :car, autosave: true, dependent: :destroy
      validates :makes, presence: true
      accepts_nested_attributes_for :makes, allow_destroy: :true, reject_if: :all_blank
      ...
    end

    class Make < ActiveRecord::Base
      ...
      has_many  :pricings, inverse_of: :make, autosave: true, dependent: :destroy
      validates :car,      presence: true
      accepts_nested_attributes_for :features, :pricings, allow_destroy: true
      ...
    end

    class Pricing < ActiveRecord::Base
      ...
      validates :make, presence: true
      ...
    end

    form.haml
    = form_for @car do |f|
      ...
      = f.fields_for :makes do |v|
        = v.hidden_field :id


**ASSIGNING DROP-DOWN SELECTIONS TO NESTED ATTRIBUTE VIA JOIN TABLE**

This is a complex issue because not only are we trying to assign two sub-sets of the same collection (color and body-type) per make, but we're attempting to do it through a join table.

What's causing this error:

    [...]Controller# (ActionView::Template::Error) "Unknown primary key for [table] in model [...]"

The reason has_many :through associations allow you to do all sorts of things through join tables that you wouldn't ordinarily be able to with HABTM (such as callbacks and validations), is because there's a primary key to reference each joined record. If all you're doing is joining tables and don't need any special behavior, then it's quickest to use HABTM and create a join table. That's what we had, but we needed more.


**JOIN MODEL ASSOCIATION: Using has_many :through on a join table**

Since we are assigning features before a make_id exists, has_many associations to the join table need to be written so as to allow parent-child validation through bi-directional association. This is done following the same logic as employed in the previous example.

Note that Rails needs a primary key to manage has_many :through associations on join tables. Since my join table already exists, I run a migration to add a primary key to the join table and use first: true to make it the first column. Finally, I update my form builder and nested_attributes on my form to reflect the association to the join table. 

Notice that the feature value for each drop-down is simultaneously built along with the join record. Instantiating one of each FeatureType when building the form: (a) takes care of the last drop-down over-writing the first, and (b) gives us a handy new join record to link through to each of our features.

    class AddIdToFeaturesMakes < ActiveRecord::Migration
      def change
        add_column :features_makes, :id, :primary_key, first: true
      end
    end

    class Make < ActiveRecord::Base
      ...
      has_many :features,       through: :features_makes, source: :feature
      has_many :features_makes, inverse_of: :make, autosave: true, dependent: :destroy
      ...
    end

    class FeaturesMakes < ActiveRecord::Base
      attr_accessible :make_id, :feature_id, :feature_attributes
      belongs_to :make
      belongs_to :feature
      validates  :make, :feature, presence: true
    end

    class Feature < ActiveRecord::Base
      ...
      has_many :makes,          through: :features_makes
      has_many :features_makes, inverse_of: :feature, autosave: true, dependent: :destroy
      ...
    end

    CarsController.rb
    def new
      @car = Car.new
      make = @car.makes.build
      make.pricings.build
      2.times { |n| make.features_makes.build.build_feature(feature_type_id: n+1) }
    end

    form.haml
    = v.fields_for :features_makes do |ov|
      - features = ::Feature.where(feature_type_id: ov.object.feature.feature_type_id)
      = ov.label :feature_id, "#{ov.object.feature.feature_type.name}"
      = ov.collection_select "feature_id", features.order(:name), :id, :name, { prompt: "Select #{ov.object.feature.feature_type.name}" }, { class: "selectpicker select-block", data: {style: "btn-primary"} }


**Why do nested attributes work on #create but not on #update?**

If for any reason you decide to leave the primary key out, it is absolutely possible to assign feature values to a new make. Through trial and error - trying to get this to work - I discovered that you can use has_many :through without adding a primary key. You can tell Rails what primary key you want it to use, which can be an array (or composite):

    class Featuresmake < ActiveRecord::Base
      attr_accessible :make_id, :feature_id, :feature_attributes
      self.primary_key = [:make_id, :feature_id] <-- [ Setting primary key on a join table ]
      ...
    end

But - while possible - what's the point? It's a waste of real estate AND you don't get the full benefit that you get with a primary key. The biggest impediment is that you won't be able to update drop-down selections. Instead you'll end up with two new drop-downs anytime you try to update color or body-type for an existing make. The reason for this is that Rails will instantiate a new record when it can't find a primary key. So, there you go. 


**FINAL VERSION: THE CODE THAT AUTOMAGICALLY WORKS**

  JOIN MODEL ASSOCIATION: Add primary key to join table

    class AddIdToFeaturesmakes < ActiveRecord::Migration
      def change
        add_column :features_makes, :id, :primary_key, first: true
      end
    end

  FORM BUILDER: Build instances for desired associations

    CarsController.new
    def new
      @car = Car.new
      make = @car.makes.build
      make.pricings.build
      2.times { |n| make.features_makes.build.build_feature(feature_type_id: n+1) }
    end

  BI-DIRECTIONAL ASSOCIATION: Specify parent-child relationship

    class Car < ActiveRecord::Base
      attr_accessible :model, :description, :model_code, :makes_attributes
      has_many  :makes,          inverse_of: :car, autosave: true, dependent: :destroy
      has_many  :feature_types,  through: :features
      has_many  :features,       through: :makes
      has_many  :pricings,       through: :makes
      validates :makes,          presence: true <--[ Validates presence of inverse_of object ]
      accepts_nested_attributes_for :makes, allow_destroy: :true, reject_if: :all_blank
      (...)
    end

    class Make < ActiveRecord::Base
      attr_accessible :name, :car_id, :model_code, :pricings_attributes, :features_attributes
      belongs_to :car,          touch: true
      has_many :pricings,       inverse_of: :make, autosave: true, dependent: :destroy
      has_many :features_makes, inverse_of: :make, autosave: true, dependent: :destroy
      has_many :images,         through: :features
      has_many :feature_types,  through: :features
      has_many :features,       through: :features_makes, source: :feature ### replaces habtm
      validates :car,           presence: true <--[ Validates presence of inverse_of object ]
      accepts_nested_attributes_for :features_makes, :pricings, allow_destroy: true
      (...)
    end

    class Pricing < ActiveRecord::Base
      attr_accessible :make_id, :currency, :price
      belongs_to :make,     touch: true
      validates  :make,     presence: true <--[ Validates presence of inverse_of object ]
      validates  :currency, presence: true
      (...)
    end

    class FeatureType < ActiveRecord::Base
      attr_accessible :name
      has_many  :features, dependent: :destroy
      validates :name, presence: true
    end

    class Feature < ActiveRecord::Base
      attr_accessible :name, :presentation, :feature_type_id
      belongs_to :feature_type
      has_many   :features_makes, inverse_of: :feature, autosave: true, dependent: :destroy
      has_many   :makes,          through: :features_makes
      validates  :name,           presence: true 
      (...)
    end

    class FeaturesMakes < ActiveRecord::Base
      attr_accessible :make_id, :feature_id, :feature_attributes
      belongs_to :make
      belongs_to :feature
      validates  :make, :feature, presence: true <--[ Validates presence of inverse_of object ]
    end

  INPUT FORM

    form.haml
    = form_for @car do |f|
      = f.label :car_model
      = f.text_field :model

      = f.label :model_code
      = f.text_field :model_code

      = f.fields_for :makes do |v|
        = v.hidden_field :id
        
        = v.label :vin
        = v.text_field :vin

        = v.fields_for :features_makes do |ov|
          - features = ::Feature.where(feature_type_id: ov.object.feature.feature_type_id)
          = ov.label :feature_id, "#{ov.object.feature.feature_type.name}"
          = ov.collection_select "feature_id", features.order(:name), :id, :name, { prompt: "Select #{ov.object.feature.feature_type.name}" }, { class: "selectpicker select-block", data: {style: "btn-primary"} }

        = v.fields_for :pricings do |p|
          = p.label :price
          = p.select :currency, Pricing.all.collect{ |p| [currency_symbol(p.currency), p.currency] }, { include_blank: "Currency" }, { class: "selectpicker", data: {style: "btn-primary currency-dropdown"} }
          = p.text_field :price

  PARAMS
    "car"=>{"name"=>"delano", "model_code"=>"dx",
      "makes_attributes"=>{"0"=>{"vin"=>"vin98765", 
        "features_makes_attributes"=>{"0"=>{"feature_id"=>"6"}, "1"=>{"feature_id"=>"4"}}, 
        "pricings_attributes"=>{"0"=>{"currency"=>"usd", "price"=>"100000"}}}}}

Sources:<br>
http://robots.thoughtbot.com/accepts-nested-attributes-for-with-has-many-through
http://api.rubyonrails.org/classes/ActiveRecord/Associations/ClassMethods.html
http://www.sitepoint.com/complex-rails-forms-with-nested-attributes
