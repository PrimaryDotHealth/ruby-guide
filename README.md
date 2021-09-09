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

Services is a wrong name for a folder that handle business actions, I think it is ok if a service class connects to an external service.

### Actions (Light service)
*See official docs [here](https://github.com/adomokos/light-service)*

Actions wrap business logic that is straight-forward. In this example, we just create a todo with an item that sends an email.

```ruby
class CreateTodoWithItem
  extends LightService::Action
  
  expects :todo_params, :first_item_body
  promises :todo, :first_item
  executed do |context|
    context.todo = Todo.new(context.todo_params)
    context.first_item = Item.new(todo: context.todo, body: context.first_item_body, active: true)
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

This is a personal opinion but I don't like Light service gem, one of the problems that I see is that everything is handled in a class method and working with class methods is hard, specially when the action grows a little bit.
So if we use Services or business actions (call it whatever you want) I think there is no need for this.

### Models
Model code should only deal with validating data and loading data. It should not try to do business logic-y stuff.

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

I agree completely about it. We shouldn't put business logic here, neither presentation logic. A model only connects to the database, of course you can use validations but usually data validation should be handled in the request context and not in the model.

### View Models
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

Why is this bad? Because `title_with_items_count` isn't *really* a model attribute, and this kind of code tends to blow up a file. Count the lines in user.rb if you don't believe me! It is also a presentation concern, not a persistence issue. If some other view wants to display this slightly differently, you would then add add a second method to the model, and so on. 

Another common practice is to put this in a Helper. 

View models or presenters are fine to hold the presentation logic, this help us to create tiny controller instead of putting a lot of code there.

```ruby
# Bad
# app/helpers/todo_helper.rb
module TodoHelper
  def title_with_items_count(todo)
    "#{todo.title} (#{todo.items.count})"
  end 
end
```

However, this is also bad because 
a) it's hard to know where `title_with_items_count` is defined
b) Someone else could define another helper with the same name in some other Helper module and it would be hard to debug
c) All helpers are mixed into every view, increasing memory pressure

Helpers are fine if something is used acroos multiple views, e.g. rails helpers to manipulate strings, url helpers, and so on. But that's all, put business logic there is a bad decision because helpers aren't namespaced, all helpers put in all modules are available in all views and you could end up with method names very large.

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

I wouldn't use drapper if we use view models, there is no need to add a new gem when you can handle presentation logic with presenter or view models.

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

  def send_email
    TodoMailer.send(self)
  end
end
```
Why is this bad? Because it becomes harder and harder to test this `Todo` model without a bunch of stuff happening.


Never use callbacks in a model unless it is strictly needed, e.g. a gem that index the model in ElasticSearch (like Chewy). The problem is that you have undesired behaviours that could not control and refactor code is very complicated.


### Modules (a.k.a. View Helpers) (probably don't make these)
Helpers are *really bad*. They kinda just dump code into a view, and it's really hard to figure out where its defined. They shoudl only be used for really stateless, generic formatting stuff. For example:
```ruby
# app/helpers/todo_helper.rb
module TodoHelper
  def format_datetime(datetime)
    datetime.strftime("%Y:%m:%d, %H:%M:%S") # Always format it like year, month, day hour, minute second
  end
end
```

If you are thinking of making a helper module, here are a few questions:
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

```ruby
# Bad
module ArticlesHelper
  def article_titles_with_id(articles)
    articles.map { |a| "#{a.id} #{a.title}" }
  end
end

# Good
class ArticlesIndexViewModel
  def initialize(articles)
    @articles = articles
  end
  
  def article_titles_with_id
    @articles.map { |a| "#{a.id} #{a.title}" }
  end
end
```

