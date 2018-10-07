---
layout: post
title:  "Introducing WatirPump"
date:   2018-10-07 09:05:15 +0200
excerpt: Proudly presenting a new, fresh look at Page Object Model for Ruby and Watir
categories:
---

So one could ask: "Why yet another PageObject library?". After all, there are some very decent gems out there already. This is true. Each of the existing libraries offers certain unique features, but none of them has all. Well, WatirPump - the new kid on the block, has the ambition to make an exception. Let's take a look at it and find out.

# Introduction
It's never easy describing code using a natural language. Especially if it's a foreign one ;) For the in-depth explanation about what `WatirPump` can do and how it does it I highly recommend reading its [README](https://github.com/bwilczek/watir_pump/blob/master/README.md).

There is also a series of specs that demonstrate certain features of the library in a form of a [tutorial](https://github.com/bwilczek/watir_pump_tutorial).

For this article, let's try to use as little English as possible. Lets let the code speak for itself.

# Example spec

Let's test a [page with multiple ToDo lists](https://bwilczek.github.io/watir_pump_tutorial/todo_lists.html). This example scenario checks if addition and removal of list items work properly.

![Example page with multiple ToDo lists](/blog/assets/watir_pump/todo_lists.png)

Having properly modeled the `ToDoListsPage` class the spec will look like this:

```ruby
RSpec.describe ToDoListsPage do
  it 'Adds and removes items' do
    ToDoListsPage.open do
      todo_lists['Groceries'].add('Pineapple')
      expect(todo_lists['Groceries']).to include 'Pineapple'

      todo_lists['Work'].add('Read RubyWeekly')
      expect(todo_lists['Work']).to include 'Read RubyWeekly'

      todo_lists['Groceries']['Bread'].remove
      expect(todo_lists['Groceries']).not_to include 'Bread'
    end
  end
end
```

Pretty neat, right?

Let's dig into the internals and learn how to build such a nice page API with WatirPump.

# ToDoList

As we look at the page it is clear that the first candidate to be extracted into a reusable component is the `ToDoList`. It contains several elements:

* a title
* a text field for a name of a new item
* a button that adds the new item of given name
* a list of the existing list items

Let's try to represent this component in a code:

```ruby
class ToDoList < WatirPump::Component
  # Let's use some WatirPump class macros (explained below)
  div_reader :title, role: 'title'
  text_field_writer :item_name, role: 'new_item'
  button_clicker :submit, role: 'add'
  components :items, ToDoListItem, :lis
  query :values, -> { items.map(&:label) }

  # and some extra methods to make the spec look more natural
  def [](label)
    items.find { |i| i.label == label }
  end

  def add(item_name)
    fill_form!(item_name: item_name)
  end

  def include?(item)
    !self[item].nil?
  end
end
```

Looks intuitive, however, there are some concepts that might require a few words of explanation.

#### Concept 0: class macros (some would call it a DSL)

Classes that inherit from `WatirPump::Page` or `WatirPump::Component` gain access to a set of powerful class macros that (behind the scenes) generate methods which interact with the webpage. For example, each DOM element can be declared with its own class macro. Just like in plain watir code.

```ruby
p :summary, role: 'summary'

# is a shorthand for:
def summary
  root.p(role: 'summary')
end
```

See WatirPump [docs](https://github.com/bwilczek/watir_pump/blob/master/README.md#elements) to learn more about how one can declare page elements (don't forget to check out how lambdas could help here). While you read the docs please also grep for `root vs browser` to learn the difference.

#### Concept 1: element action macros: reader, writer, clicker

It happens quite often, that there is no point in declaring a "plain" HTML element in the page class. After all an abstract DOM node is rather useless from the perspective of a functional test API. What matters, however, is what the user could do with it. And the user usually needs to perform a certain action:

* read the value of the element
* write a new value to the element
* click the element

This is where element action class macros come in: they generate methods that perform actions on the element with given locator.

```ruby
div_reader :title, role: 'title'

# is a shorthand for:
def title
  root.div(role: 'title').text
end

text_field_writer :item_name, role: 'new_item'

# is a shorthand for:
def item_name=(val) # mind the '=' !
  root.text_field(role: 'new_item').set(val)
end

button_clicker :submit, role: 'add'

# is a shorthand for
def submit
  root.button(role: 'add').click
end
```

See the [documentation about element action macros](https://github.com/bwilczek/watir_pump/blob/master/README.md#element-action-macros-1) for more details.

#### Concept 2: components class macro

Declares a collection of components of given class that are located within the current component using the given locator.

The line below declares a ToDoListItem component instance at every li element

```ruby
components :items, ToDoListItem, :lis
```

#### Concept 3: fill_form!

Instance method `fill_form` is a fast way to invoke all writers declared for the given component. In our case there is only one: `item_name=`. `fill_form` accepts a hash of writer method names and values for them. If our component contains a `submit` method `fill_form!` (notice the !) will try to invoke it after all writers are executed.

So to sum it up: `fill_form!(item_name: 'Pineapple')` will do `self.item_name='Pineapple' ; submit`

The more writers the component has, the more work can `fill_form` do for us.

For more information about form helpers please [see the docs](https://github.com/bwilczek/watir_pump/blob/master/README.md#form-helpers).

#### The extra methods: add, [], include?

These are rather self-explanatory. They are here to let us write more expressive code. Like this:

```ruby
todo_list.add('Pineapple')
todo_list.include? 'Pineapple'
todo_list['Pineapple'].remove
```

#### Concept 4: query macro

There is one more concept that has been employed in this example. It's a `query` class macro and more information about it can be found [here](https://github.com/bwilczek/watir_pump/blob/master/README.md#query-class-macro).

# ToDoListItem

An item consists of two elements:

* a label
* a link to remove the item

When represented as a code they would look like this:

```ruby
class ToDoListItem < WatirPump::Component
  link_clicker :remove, role: 'rm'
  span_reader :label, role: 'name'
end
```

Both of the macros used here have been already discussed.

# ToDoListsPage

The highest object in the model hierarchy is the `ToDoListsPage`. Its declaration doesn't differ much from the declaration of a component. The main change is the presence of the required `uri` macro invocation.

Another new contstruct (macro) presented in the listing below is `decorate`. It is used to, well, decorate given method with some additional behavior. Here, the value returned by method `todo_lists` will be wrapped by an instance of a `CollectionIndexedByTitle` class. It is required to replace the default behavior of `[]` operator: from integer based (like in `Array`) to  string based (like in `Hash`). This will provide us with access to certain `ToDoLists` by their title.

```ruby
class ToDoListPage < WatirPump::Page
  uri '/todo_lists.html'
  components :todo_lists, ToDoList, -> { root.divs(role: 'todo_list') }
  decorate :todo_lists, CollectionIndexedByTitle
end
```

# CollectionIndexedByTitle

Decorator class for the collection of `ToDoLists`. As described above, all it does is overriding the default behavior of `[]` method.

```ruby
class CollectionIndexedByTitle < WatirPump::ComponentCollection
  def [](title)
    find { |l| l.title == title }
  end
end
```

# Execution with rspec

Having the page and component classes in place let's take another look at API exposed to rspec:

```ruby
RSpec.describe ToDoListsPage do
  it 'Adds and removes items' do
    # open method navigates to the page
    # then executes the provided block in the scope of page class singleton instance:
    # e.g. todo_lists['A'] == ToDoListPage.instance.todo_lists['A']
    ToDoListsPage.open do
      # todo_lists collection with decoration supports indexing with title
      # add method is invoked on a ToDoList with title 'Groceries'
      todo_lists['Groceries'].add('Pineapple')

      # 'include?' method defined in ToDoList class
      # is used by the rspec matcher 'include'
      expect(todo_lists['Groceries']).to include 'Pineapple'

      todo_lists['Work'].add('Read RubyWeekly')
      expect(todo_lists['Work']).to include 'Read RubyWeekly'

      # ToDoList supports '[]' method that is used below
      # to access ToDoListItem by its label
      todo_lists['Groceries']['Bread'].remove
      expect(todo_lists['Groceries']).not_to include 'Bread'
    end
  end
end
```

# Summary

This article showed only a few of the features that WatirPump offers to build maintainable, reusable and elegant Page Object models. For a complete guide please refer to projects [README](https://github.com/bwilczek/watir_pump/blob/master/README.md) and a [tutorial](https://github.com/bwilczek/watir_pump_tutorial).

In case of ideas for improvement, found bugs or any other questions don't hesitate to:

* [create a pull request](https://github.com/bwilczek/watir_pump/pulls) with your feature
* [report an issue](https://github.com/bwilczek/watir_pump/issues)
* [contact the author via email](mailto:bwilczek@gmail.com)
