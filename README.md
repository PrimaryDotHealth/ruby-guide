# Ruby Guide

We'll be mocking up a todo app for our example.

**Models**:

`Todo(id: int, title: string, owner_email: string)`

`Item(id: int, todo_id: int, body: string, active: boolean)`

For example, the data could be something like:

```
Todo(1, "Weekend Todo List", "name@example.com")
Item(1, 1, "Do laundry", true)
Item(2, 1, "Walk the dog", true)
Item(3, 1, "Go for a run", false)
```

**Business Logic**:
*Business logic is just rules of how your app behaves. Here are some we will require from our app:*

* A todo must have at least one item (you can't have an empty todo list). *This means you have to create a todo list with at least one thing on it*
* Whenever an item is added to an existing todo, send an email to the owner.
* Whenever a todo is created, send an email to the owner.

## Where to put code?

### Services
Services wrap business logic that is complex. In this example, you have a `all_valid?` and `run!` method, that is dependent on an instance variable `@todo_data`.

```ruby
class TodosImporter
  def initialize(todos_data)
    @todos_data = todos_data
  end
  
  def all_valid?
    @todos_data.all? { |u| is_valid?(u) }
  end

  def run!
    @todos_data.each do |todo_data|
      Todo.create!(todo_data)
    end
  end
end
```

### Actions (Light service)
*See official docs [here](https://github.com/adomokos/light-service)*


Actions wrap business logic that is straight-forward. In this example, we just create a todo with an item that sends an email

```ruby
class CreateTodoWithItem
  extends LightService::Action
  
  expects :todo_params, :first_item_body
  promises :todo, :first_item
  executed do |context|
    context.todo = Todo.new(context.todo_params)
    context.item = Item.new(todo: context.todo, body: context.first_item_body, active: true)
    if !context.todo.save
      context.fail!("Failed to save")
    end
  end
end

class SendCreatedEmailToOwner
  extends LightService::Action
  
  expects :todo
  executed do |context|
    TodoMailer.deliver_now(todo, "Todo Created!")
  end
end

class CreateTodo
  extends LightService::Organizer
  def self.call(todo_params, first_item_body)
    with(todo_params: todo_params, first_item_body: first_item_body).reduce([CreateTodoWithItem, SendCreatedEmailToOwner])
  end
```

### Models
Model code should only deal with validating data and loading data. It should not try to do business logic-y stuff
```ruby
class Todo < ApplicationRecord
  has_many :items

  validates_presence_of :title
end

class Item < ApplicationRecord
  belongs_to :todo

  validates_presence_of :body
end

# Bad

class Todo < ApplicationRecord
  has_many :items

  validates_presence_of :title
  
  # This is the boundary between two models, it should be done in a action or service
  def create_item(body)
    Item.create!(todo: self, body: body)
  end
end
```

### View Models
```
```
View Models are a way to group view specific helpers.

For example, maybe we want to render the title of our todo like: "{title} ({number_of_active_items})"

```ruby
# Good
class TodoViewHelper
  def initialize(todo)
    @todo = todo
    @items = todo.items
  end

  def title_with_items_count
    "#{@todo.title} (#{@items.count})"
  end
end

# To use this
@view_model = TodoViewModel.new(todo: @todo)

# Bad

class Todo < ApplicationRecord
  def title_with_items_count
    "#{title} (#{items.count})"
  end
end

```

Why is this bad? Because `title_with_items_count` isn't *really* a model attribute, and this kind of code tends to blow up a file. Count the lines in user.rb if you don't believe me!


### Decorators (Draper) (new!)

*See official docs [here](https://github.com/drapergem/draper)*

Draper will be used to decorate model with attributes that aren't really useful for business logic, but are useful for views.

For example, maybe I want something like `item.active_string` which returns either `"Active"` or `"Inactive"`
```ruby
# Good

class TodoDecorator < Draper::Decorator
  delegate_all

  def active_string
    if active?
      "Active (Last updated #{updated_at_time})"
    else
      "Inactive"
    end
  end

  def updated_at_time
    updated_at.strftime("%A, %B %e")
  end
end


# Bad: Putting it directly into the model

class Todo < ApplicationRecord
  def active_string
    if active?
      "Active (Last updated #{updated_at_time})"
    else
      "Inactive"
    end
  end

  def updated_at_time
    updated_at.strftime("%A, %B %e")
  end
end
```

### Model Hooks
Most of the rails community has turned against using hooks, you probably shouldn't use them except as validators
```ruby
class Todo < ApplicationModel
  
  validate :validate_title_is_uppercase

  def validate_title_is_uppercase
    if self.title.first != self.title.first.upcase
      errors.add(:title, "not uppercased")
    end
  end
end

# Bad

class Todo < ApplicationModel
  after_save :send_email

  def validate_title_is_uppercase
    TodoMailer.send(self)
  end
end
```
Why is this bad? Because it becomes harder and harder to test this `Todo` model without a bunch of stuff happening.

### Modules (probably don't make these)
Modules are *really bad*. They kinda just dump code into a controller, and it's really hard to figure out where its defined. They shoudl only be used for really stateless, generic formatting stuff. For example:
```ruby
class TodoModule
  def format_datetime(datetime)
    datetime.strftime("%Y:%m:%d, %H:%M:%S) # Always format it like year, month, day hour, minute second
  end
end
```

If you are thinking of making a module, here are a few questions:
1. Is it reading from instance variables? Please just pass them in as params
```ruby
# Bad
def todos_count_minus_one
  @todos.count - 1
end

# Good
def todos_count_minus_one(todos)
  todos.count - 1
end
```
2. Is it specific to a view? Just put them into view models!


