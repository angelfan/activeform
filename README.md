# ActiveForm

Set your models free from the `accepts_nested_attributes_for` helper. ActiveForm provides an object-oriented approach to represent your forms by building a Form Object, rather than relying on ActiveRecord internals for doing this. The Form Object provides an API to describe the models involved in the form, their attributes and validations. The Form Object deals with create/update actions of nested objects in a more seamless way.

## Installation

Add this line to your Gemfile:


```ruby
gem 'active_form'
```

## Defining Forms

Consider an example where you want to create/update a Conference that can have many Speakers which can present a single Presentation with one form submission. You start by defining a form to represent the root model, Conference.

```ruby
class ConferenceForm < ActiveForm::Base
  self.main_model = :conference
  
  attributes :name, :city
  
  validates :name, :city, presence: true
end
```

Your Form Object has to subclass `ActiveForm::Base` in order to gain the necessary API. When defining the form, you have to specify the main_model the form represents with the following line:
```ruby
self.main_model = :conference
```
To add fields to the form, use the `::attributes` or `::attribute` method. The form can also define validation rules for the model it represents. For the `presence` validation rule there is a short inline syntax:

```ruby
class ConferenceForm < ActiveForm::Base
  attributes :name, :city, required: true
end
```

## The API

The ActiveForm::Base class provides a simple API with only a few instance/class methods. Below are listed the instance methods:

1. `#initialize(model)` accepts an instance of the model that the form represents.
2. `#submit(params)` updates the main form's model and nested models with the posted parameters. The models are not saved/updated until you call `#save`.
3. `#errors` returns validation messages in a classy ActiveModel style.
4. `#save` will call `#save` on the model and nested models. This method will validate the model and nested models and if no error arises then it will save them and return true.

The following are the class methods:

1. `::attributes` accepts the names of attributes to define on the form. If you want to declare a presence validation rule for the given attributes, you can pass in the `required: true` option as showcased above. The `::attribute` method is aliased to the `::attributes` method.
2. `::association(name, options={}, &block)` defines a nested form for the `name` model. If the model is a `:has_many` association you can pass in the `records: x` option and fields to create `x` objects will be rendered. If you pass a block, you can define another nested form with the same way.

In addition to the main API, forms expose accessors to the defined attributes. This is used for rendering or manual operations.

## Setup

In your controller you create a form instance and pass in the model you want to work on.

```ruby
class ConferencesController
  def new
    @form = ConferenceForm.new(Conference.new)
  end
```

You can also setup the form for editing existing items.

```ruby
class ConferencesController
  def edit
    @form = ConferenceForm.new(Conference.find(1))
  end
```

ActiveForm will read property values from the model in setup. Given the following form class.

```ruby
class ConferenceForm < ActiveForm::Base
  attribute :name
```

Internally, this form will call `conference.name` to populate the name field.

## Rendering Forms

Your `@form` is now ready to be rendered, either do it yourself or use something like Rails' `#form_for`, `simple_form` or `formtastic`.

```haml
= form_for @form do |f|

  = f.input :name
  = f.input :city
```

Nested forms and collections can be easily rendered with `fields_for`, etc. Just use ActiveForm as if it would be an ActiveModel instance in the view layer.

## Syncing Back

After setting up your Form Object, you can populate the models with the submitted parameters.

```ruby
class ConferencesController
  def create
    @form = ConferenceForm.new(Conference.new)
    @form.submit(conference_params)
  end
```

This will write all the properties back to the model. In a nested form, this works recursively, of course.

## Saving Forms

After the form is populated with the posted data, you can save the model by calling #save.

```ruby
class ConferencesController
  def create
    @form = ConferenceForm.new(Conference.new)
    @form.submit(conference_params)

    respond_to do |format|
      if @form.save
        format.html { redirect_to @form, notice: "Conference: #{@form.name} was successfully created." }
      else
        format.html { render :new }
      end
    end
  end
end
```

If the #save method returns false due to validation errors defined on the form, you can render it again with the data that has been submitted and the errors found.

## Nesting Forms: 1-n Relations

ActiveForm also gives you nested collections.

Let's define the `has_many :speakers` collection association on the Conference model.

```ruby
class Conference < ActiveRecord::Base
  has_many :speakers
  validates :name, uniqueness: true
end
```

The form should look like this.

```ruby
class ConferenceForm < ActiveForm::Base
  attributes :name, :city, required: true

  association :speakers do
    attributes :name, :occupation, required: true
  end
end
```

By default, the `association :speakers` declaration will create a single Speaker object. You can specify how many objects you want in your form to be rendered with the `new` action as follows: `association: speakers, records: 2`. This will create 2 new Speaker objects, and ofcourse fields to create 2 Speaker objects. There are also some link helpers to dynamically add/remove objects from collection associations. Read below.

This basically works like a nested `property` that iterates over a collection of speakers.

### has_many: Rendering

ActiveForm will expose the collection using the `#speakers` method.

```haml
= form_for @form |f|
  = f.text_field :name
  = f.text_field :city

  = f.fields_for :speakers do |s|
    = s.text_field :name
    = s.text_field :occupation
```

## Nesting Forms: 1-1 Relations

Speakers are allowed to have 1 Presentation.

```ruby
class Speaker < ActiveRecord::Base
  has_one :presentation
  belongs_to :conference
  validates :name, uniqueness: true
end
```

The full form should look like this:

```ruby
  class ConferenceForm < ActiveForm::Base
  attributes :name, :city, required: true

  association :speakers do
    attribute :name, :occupation, required: true

    association :presentation do
      attribute :topic, :duration, required: true
    end
  end
end
```

### has_one: Rendering

Use `#fields_for` in a Rails environment to correctly setup the structure of params.

```haml
= form_for @form |f|
  = f.text_field :name
  = f.text_field :city
  
  = f.fields_for :speakers, records: 2 do |s|
    = s.text_field :name
    = s.text_field :occupation
    
    = f.fields_for :presentation do |p|
      = p.text_field :topic
      = p.text_field :duration
```