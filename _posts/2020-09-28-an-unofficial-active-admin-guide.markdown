---
layout: post
title:  "An Unofficial Active Admin Guide"
date:   2020-09-24 22:32:11 +0300
categories: rails activeadmin
---

Recently I bumped into **Rails Survey 2020** results and saw [the top 10 gems frustrate one the most](https://rails-hosting.com/2020/#which-ruby-gems-frustrate-you-the-most).
On the 5th place, there was the **Active Admin** gem. I would not say this was an unexpected result.
I often come across the opinion that Active Admin is only suitable for a 15-minute blog, but there
is much more with this library.

Here are some approaches my colleagues and I take when working with Active Admin.

![An Unofficial Active Admin Guide](/assets/an-unofficial-active-admin-guide/main.jpg)

Active Admin is based on several libraries, among which I would highlight `arbre`, `formtastic`,
`inherited_resources`, and `ransack`. Each of them is responsible for its part and deserves separate
consideration. Let's start alphabetically with the library extracted from Active Admin itself.

## Arbre: custom components

One of the problems with Active Admin is rapidly growing resource files: filters, additional
actions, templates, forms, and so on – everything is in one file. I can hear a lonely moan somewhere
in the distance: "what about the single responsibility principle?" There is none. Let me show you
how you can isolate some templates in separate classes.

**Arbre** is a library for defining templates using Ruby objects. Here's an example of a basic page
written with Arbre DSL:

{% highlight ruby %}
html do
 head do
   title('Welcome page')
 end
 body do
   para('Hello, world')
 end
end
{% endhighlight %}

DSL can be extended with components. For example, in Active Admin, these are `tabs`, `table_for`,
`paginated_collection`, and even resource pages themselves.  Next, we'll dive in and explore the structure
of the basic Arbre component.

### Arbre: hello world component

Like all Arbre components, our `Admin::Components::HelloWorld` inherits from
[`Arbre::Component`](https://github.com/activeadmin/arbre/blob/master/lib/arbre/component.rb) class:

{% highlight ruby %}
# app/admin/components/hello_world.rb
module Admin
  module Components
    class HelloWorld < Arbre::Component
      builder_method :hello_world

      def build(attributes = {})
        super(attributes)
        text_node('Hello world!')
        add_class('hello-world')
      end

      def tag_name
        'h1'
      end
    end
  end
end
{% endhighlight %}

Starting from the top: `builder_method` defines a method to create a component using DSL.
Arguments passed to the component will be passed to the `#build` method.

Each Arbre component is a separate DOM element (similar to the way modern frontend frameworks work,
only dates back to 2012). All components are rendered as `div` DOM elements by default. You can
override `#tag_name` method to change this behavior. As you might guess, `#add_class` method adds a
`class` attribute to the root DOM element.

At this point, the only thing left is to call our new component.
For example, let's do this in `app/admin/dashboard.rb`:

{% highlight ruby %}
# app/admin/dashboard.rb
ActiveAdmin.register_page 'Dashboard' do
  menu priority: 1, label: proc { I18n.t('active_admin.dashboard') }

  content do
    hello_world
  end
end
{% endhighlight %}

![Hello world component](/assets/an-unofficial-active-admin-guide/01-hello-world-component.jpg)

Up next is an example of a small refactoring of the admin panel using a custom component.

### Arbre: (almost) a real-life example

To understand how to use Arbre in a production environment, let's assume that we have a blog with
posts (`Post`) and comments (`Comment`) with a 1:M relationship. We need to display the last ten
comments on the `show` page of a post.

{% highlight ruby %}
# app/admin/posts.rb
ActiveAdmin.register Post do
  permit_params :title, :body

  show do
    attributes_table(:body, :created_at)

    panel I18n.t('active_admin.posts.new_comments') do
      table_for resource.comments.order(created_at: :desc).first(10) do
        column(:author)
        column(:text)
        column(:created_at)
      end
    end
  end
end
{% endhighlight %}

![New comments component](/assets/an-unofficial-active-admin-guide/02-new-comments.jpg)

Now we'll move the table with comments into a separate component. Create a new class and inherit it
from [`ActiveAdmin::Views::Panel`](https://github.com/activeadmin/activeadmin/blob/master/lib/active_admin/views/components/panel.rb).
If you create a new component from scratch (as in `hello_world` example above) and call `panel` from
it, `panel` will be wrapped by another `div`, and this will probably break the layout.

Put our new class in `app/admin/components/posts/new_comments.rb`, since Active Admin automatically
requires everything inside `app/admin/**/*`:

{% highlight ruby %}
# app/admin/components/posts/new_comments.rb
module Admin
  module Components
    module Posts
      class NewComments < ActiveAdmin::Views::Panel
        builder_method :posts_new_comments

        def build(post)
          super(I18n.t('active_admin.posts.new_comments'))
          table_for last_comments(post) do
            column(:author)
            column(:text)
            column(:created_at)
          end
        end

        private

        def last_comments(post)
          post.comments
              .order(created_at::desc)
              .first(10)
        end
      end
    end
  end
end
{% endhighlight %}

Replace `panel` in `app/admin/posts.rb` with our new component
and pass `resource` object as an argument:

{% highlight ruby %}
# app/admin/posts.rb
ActiveAdmin.register Post do
  permit_params :title, :body

  show do
    attributes_table(:body, :created_at)
    posts_new_comments(resource)
  end
end
{% endhighlight %}

Awesome! Note that `resource` is also available from the component's context. However, by explicitly
passing `resource` to the builder, we achieve loose coupling, which allows us to reuse the component
in the future.

Speaking of reuse, we can extract everything from the `show` block (as well as other template
blocks) into partial:

{% highlight ruby %}
# app/admin/posts.rb
ActiveAdmin.register Post do
  show do
    render('show', post: resource)
  end
end
{% endhighlight %}

{% highlight ruby %}
# app/views/admin/posts/_show.html.arb
panel(ActiveAdmin::Localizers.resource(active_admin_config).t(:details)) do
  attributes_table_for(post, :body, :created_at)
end

posts_new_comments(post)
{% endhighlight %}

**Note:** you can use familiar `.erb` and other templating engines instead of `.arb`.

### Arbre: what's next

First of all, I do advise you to read Active Admin components'
[official documentation](https://activeadmin.info/12-arbre-components.html).

Besides, you can read code for [base components from `arbre`](https://github.com/activeadmin/arbre/blob/master/lib/arbre/element.rb)
and [`activeadmin` components](https://github.com/activeadmin/activeadmin/tree/master/lib/active_admin/views/components).
Those are the components your custom ones will inherit from. Also, check out the gem
[`activeadmin_addons`](https://github.com/platanus/activeadmin_addons) – it has a lot of interesting
custom components.

Well, if you still write code with errors (this is still a thing for some reason), check out how to
[test custom components](https://github.com/activeadmin/activeadmin/blob/master/spec/unit/views/components/panel_spec.rb).

## Formtastic: custom forms

Formtastic is a library for describing forms using DSL. The simplest form looks like this:

{% highlight ruby %}
semantic_form_for(object) do |f|
  f.inputs
  f.actions
end
{% endhighlight %}

In the example, Formtastic automatically extracts all attributes from the passed `object` and
inserts them into a form with default input types. A list of available input types can be found in
the [README](https://github.com/formtastic/formtastic#the-available-inputs). Like Arbre, Formtastic
can be extended by creating custom component classes. To understand the basics, we'll create
a hello world component.

### Formtastic: hello world component

By analogy with Arbre components, we'll place the new class in `app/admin/inputs`:

{% highlight ruby %}
# app/admin/inputs/hello_world_input.rb
class HelloWorldInput
  include Formtastic::Inputs::Base
  def to_html
    "Input for ##{object.public_send(method)}"
  end
end
{% endhighlight %}

To apply a new input type, simply specify its name as `:as` parameter, for example:

{% highlight ruby %}
# app/admin/posts.rb
ActiveAdmin.register Post do
  form do |f|
    f.inputs do
      f.input(:id, as: :hello_world)
      f.input(:title)
      f.input(:body)
    end
    f.actions
  end
end
{% endhighlight %}

![Hello world input](/assets/an-unofficial-active-admin-guide/03-hello-world-input.jpg)

All the parameters required to draw the form (including `object` and `method`) are passed to
`#initialize` defined in the module [`Formtastic::Inputs::Base`](https://github.com/formtastic/formtastic/blob/master/lib/formtastic/inputs/base.rb).
The `#to_html` method is responsible for rendering the input.

The example may seem useless, but in fact, we use it to render read-only fields. To turn our hello
world to a useful read-only input, we only need to add a couple of methods from
`Formtastic::Inputs::Base`:

{% highlight ruby %}
# app/admin/inputs/hello_world_input.rb
class HelloWorldInput
  include Formtastic::Inputs::Base

  def to_html
    input_wrapping do
      label_html <<
        template.format_attribute(object, method)
    end
  end
end
{% endhighlight %}

![Read-only input](/assets/an-unofficial-active-admin-guide/04-read-only-input.jpg)

`input_wrapping` from [`Formtastic::Inputs::Base::Wrapping`](https://github.com/formtastic/formtastic/blob/master/lib/formtastic/inputs/base/wrapping.rb)
module is responsible for input wrapping and rendering error output and hints. `label_html`
from [`Formtastic::Inputs::Base::Labelling`](https://github.com/formtastic/formtastic/blob/master/lib/formtastic/inputs/base/labelling.rb)
module renders the label for input. These two helpers instantly turn our hello world into
a combat-applicable input. Last but not least, `format_attribute` from
[`ActiveAdmin::ViewHelpers::DisplayHelper`](https://github.com/activeadmin/activeadmin/blob/master/lib/active_admin/view_helpers/display_helper.rb)
is a helper method for rendering input values.

Now we'll move to a slightly more complex example that demonstrates how to integrate a JavaScript
library with a form.

### Formtastic: (almost) a real-life example

We'll take another made-up example to demonstrate how to work with HTML, CSS, and JS. In other
words, it will cover all the steps of writing a new input.

Say we have received a request from our blog editor: when writing a post, he would like to see
the number of words directly in the input form. As you know, the world of JavaScript has libraries
for everything, and there is one for our task too: [Countable.js](https://github.com/RadLikeWhoa/Countable).
Let's take the standard input for text (textarea) and extend it with a word counter.

To implement the new input, we need:

- take the existing text input and add `div` to it to output the number of words;
- add CSS styles for the new `div`;
- initialize Countable.js and use it to write the number of words in the new `div`.

Firstly, we need to create a new class and inherit it from
[`Formtastic::Inputs::TextInput`](https://github.com/formtastic/formtastic/blob/master/lib/formtastic/inputs/text_input.rb).
Add attribute `class="countable-input"` to the element `textarea` and define a new empty `div` with
the attribute `class="countable-content"` next to it:

{% highlight ruby %}
# app/admin/inputs/countable_input.rb
class CountableInput < Formtastic::Inputs::TextInput
  def to_html
    input_wrapping do
      label_html <<
        builder.text_area(method, input_html_options.merge(class: 'countable-input')) <<
        template.content_tag(:div, '', class: 'countable-content')
    end
  end
end
{% endhighlight %}

Now, have a look at what we have added. `input_html_options` is
[a parent class](https://github.com/formtastic/formtastic/blob/master/lib/formtastic/inputs/text_input.rb)
method, which returns HTML attributes for the input.
`builder` - is an instance of the class
[`ActiveAdmin::FormBuilder`](https://github.com/activeadmin/activeadmin/blob/master/lib/active_admin/form_builder.rb),
inherited from
[`ActionView::Helpers::FormBuilder`](https://github.com/rails/rails/blob/master/actionview/lib/action_view/helpers/form_helper.rb).
`template` is the context in which the templates are executed (basically, a huge set of
view-helpers). Thus, if we need to create a piece of form, we'll call `builder`. While if we want
to use something like `link_to`, `template` will help us.

Let's call the Countable.js library: put it into `vendor/assets/javascripts`
directory and add a simple `.js` file which will call Countable.js and throw the information into
`div.countable-content` (please don't be harsh on this piece of spaghetti code):

{% highlight javascript %}
// app/assets/javascripts/inputs/countable_input.js
//= require countable.min.js

const countable_initializer = function () {
  $('.countable-input').each(function (i, e) {
    Countable.on(e, function (counter) {
      console.log( $(e).closest('.countable-content') )
      $(e).parent().find('.countable-content').html('words: ' + counter['words']);
    });
  });
}

$(countable_initializer);
$(document).on('turbolinks:load', countable_initializer);
{% endhighlight %}

Now we include it in `app/assets/javascripts/active_admin.js`:

{% highlight javascript %}
// app/assets/javascripts/active_admin.js
// ...

//= require inputs/countable_input
{% endhighlight %}

The last step is to add a CSS file and include it in `app/assets/stylesheets/active_admin.scss`:

{% highlight sass %}
// app/assets/stylesheets/inputs/countable_input.scss
.countable-content {
  float: right;
  font-weight: bold;
}
{% endhighlight %}

{% highlight sass %}
// app/assets/stylesheets/active_admin.scss
//...
@import "inputs/countable_input";
{% endhighlight %}

That's it! Our new input is ready. Now we can call it in the form:

{% highlight ruby %}
# app/admin/posts.rb
ActiveAdmin.register Post do
  form do |f|
    f.inputs do
      f.input(:id, as: :hello_world)
      f.input(:title)
      f.input(:body, as: :countable)
    end
    f.actions
  end
end
{% endhighlight %}

![Custom input](/assets/an-unofficial-active-admin-guide/05-custom-input.jpg)

This is how one can make custom components for forms, like file loaders or inputs with tricky
autofill. There is a bit more code in such components, but the approach remains the same.

### Formtastic: Warmest greetings to the DRY principle

As with the Arbre components, forms can be put into partial's, although the syntax is slightly
different:

{% highlight ruby %}
# app/admin/posts.rb
ActiveAdmin.register Post do
  form(partial: 'form')
end
{% endhighlight %}

{% highlight ruby %}
# app/views/admin/posts/_form.html.arb
active_admin_form_for(resource) do
  inputs(:title, :body)
  actions
end
{% endhighlight %}

The disadvantage of the approach is that forms are placed somewhere deep in the `views` directory.
In my opinion, this makes code navigation a bit more complicated, but that is a matter of taste.

### Formtastic: what's next

Formtastic is a pretty big library, and I would highly recommend reading the detailed
[README](https://github.com/formtastic/formtastic) to get acquainted with all customization options.
It will also be useful to see the already mentioned
[`activeadmin_addons`](https://github.com/platanus/activeadmin_addons) gem. There are lots of
additional inputs in this library that are worth being checked out.

Needless to say, although I have divided Formtastic and Arbre into different blocks of the article,
they perfectly work together. You can even create forms or parts of forms as Arbre-components.

## Inherited Resources: custom controllers

To understand where does magical `resource` come from, how to change the saving behavior, and much
more, we need to get acquainted with another gem.

**Inherited Resources** is a library designed to reduce the amount of CRUD boilerplate in
controllers.

On the one hand, the library is pretty simple, but on the other hand, it is quite comprehensive. So
let's have a quick look at a few useful methods:

{% highlight ruby %}
class PostsController < InheritedResources::Base
  respond_to :html
  respond_to :json, only: :index
  actions :index, :new, :create

  def update
    resource.updated_by = current_user
    update! { posts_path }
  end
end
{% endhighlight %}

`.respond_to` is responsible for the available formats. All `.respond_to` calls are stacked rather
than overriding each other. To reset the formats we need the `.clear_respond_to` method.

`.actions` defines available CRUD methods (`index`, `show`, `new`, `edit`, `create`, `update`,
and `destroy`).

`resource` is one of the available helpers:

{% highlight ruby %}
resource        #=> @post
collection      #=> @posts
resource_class  #=> Post
{% endhighlight %}

Finally, `#update!` is just `alias` for `#update` which can be used instead of `super` when
overloading methods.

Next, we'll have a look at the `.has_scope` method in action. Let's presume that the `post` class
has a defined scope `:published`:

{% highlight ruby %}
class Post < ApplicationRecord
  scope :published, -> { where(published: true) }
end
{% endhighlight %}

In this case, we can use the `.has_scope` method in the controller:

{% highlight ruby %}
class PostsController < InheritedResources::Base
  has_scope :published, type: :boolean
end
{% endhighlight %}

`.has_scope` adds filtering using query-parameters. In the given example, we can apply the scope
`:published` by viewing the collection at URL `/posts?published=true`.

A detailed description of these and other features of the library can be reached at the rich
[README](https://github.com/activeadmin/inherited_resources). I say we stop here and finally move to
the interaction with Active Admin.

### Inherited Resources: controller modifications

All Active Admin controllers are inherited from
[`InheritedResources::Base`](https://github.com/activeadmin/inherited_resources/blob/master/app/controllers/inherited_resources/base.rb),
which means that we can modify their behavior using library methods. For example, here is how the
list of available controller actions is defined:

{% highlight ruby %}
# app/admin/posts.rb
ActiveAdmin.register Post do
  actions :all, :except => [:destroy]
end
{% endhighlight %}

Great, we removed `delete` action from the article. It seems to be obvious: we use the
Active Admin resource as a controller. But let's not jump to conclusions yet and try to add another
feature.

By default, Active Admin includes a rendering of all pages as HTML, JSON, and XML (`index` is also
available in CSV format). Let's try to get rid of XML rendering for our page using methods that
we've already learned:

{% highlight ruby %}
# app/admin/posts.rb
ActiveAdmin.register Post do
  clear_respond_to
  respond_to :html, :json
  respond_to :csv, only: :index
end
{% endhighlight %}

![NameError](/assets/an-unofficial-active-admin-guide/06-name-error.jpg)

Oh, now we got an error `undefined method 'clear_respond_to' for #<ActiveAdmin::ResourceDSL>`.

{% highlight ruby %}
# app/admin/posts.rb
ActiveAdmin.register Post do
  controller do
    clear_respond_to
    respond_to :html, :json
    respond_to :csv, only: :index
  end
end
{% endhighlight %}

Voila, now `localhost:3000/admin/posts.xml` returns an error. And what about modifying the action's
behavior?

### Inherited Resources: method overloading

Assume that when saving we need to set the attribute `post#created_by_admin`. To do this, we'll take
advantage of the `#create` method overloading feature:

{% highlight ruby %}
# app/admin/posts.rb
ActiveAdmin.register Post do
  controller do
    def create
      build_resource
      @post.created_by_admin = true
      create!
    end
  end
end
{% endhighlight %}

We call `build_resource`, a method that initializes a new object and assigns it to the `@post`
variable. Next, set the attribute `created_by_admin` and call `create!` (aka `super`) which
continues to operate on the `@post` variable we created.

**Note:** be careful with the helpers. Inherited Resources actively uses instance variables. In the
example above, it helped us to create and modify the object, but when used carelessly, the results
may be unexpected (I learned that the hard way).

Now let's take a few steps back to the point where we've turned off XML rendering of articles.
What if we want to remove XML rendering from all the resources? We wouldn't write the same code in
every new resource, would we?

### Inherited Resources: base controller modifications

No, we wouldn't! Let's create a module that will adjust the `ActiveAdmin::ResourceController` class
behavior:

{% highlight ruby %}
# lib/active_admin/remove_xml_rendering_extension.rb
module ActiveAdmin
  module RemoveXmlRenderingExtension
    def self.included(base)
      base.send(:clear_respond_to)
      base.send(:respond_to, :html, :json)
      base.send(:respond_to, :csv, only: :index)
    end
  end
end
{% endhighlight %}

An extensible class will be passed to the `.included` method, with all needed modifications applied.
We will use the Active Admin initializer and connect the new module to
[`ActiveAdmin::ResourceController`](https://github.com/activeadmin/activeadmin/blob/master/lib/active_admin/resource_controller.rb):

{% highlight ruby %}
# config/initializers/active_admin.rb
require 'lib/active_admin/remove_xml_rendering_extension'

ActiveAdmin::ResourceController.send(
  :include,
  ActiveAdmin::RemoveXmlRenderingExtension
)
# ...
{% endhighlight %}

A bit of metaprogramming magic with `#include` and `#included`, and here you go! Now no resource
would respond to the `.xml` format.

By the way, if you think that `#prepend`, `#include`, and `#extend` methods are only useful to pass
tricky interview questions, that's quite wrong. When it is necessary to modify the code of the
external library, such approaches are often the only available tool.

### Inherited Resources: what's next

First of all, take a good look at the detailed [README](https://github.com/activeadmin/inherited_resources).
In addition, pay attention to how the controllers are organized in Active Admin, notice
the authorization logic, and other little things like additional helpers.

## Ransack: custom filters

By default, on each index page, Active Admin provides a powerful block with filtering, from which I
often have to remove something rather than add something new. But this filtering block is just the
tip of the iceberg called Ransack.

**Ransack** – a library for creating search forms, which allows you to build complex SQL queries by
interpreting the passed parameter names. It sounds complicated, but I'm sure the example will
quickly give you an understanding of what I am talking about.

For example, suppose that we need to filter blog posts (`post`) by a substring in the title
(`title`). With Ransack we can do so like this:

{% highlight ruby %}
Post.ransack(title_cont: 'sharp knives').result
{% endhighlight %}

The postfix `_cont` is one of the many predicates available in Ransack. Predicates determine which
SQL query is to be generated for search. You can read more about all available predicates in the
official [wiki](https://github.com/activerecord-hackery/ransack/wiki/Basic-Searching).

Now let's make it a bit more complicated: a customer asked us to add a filter that would allow
searching by substring of title and/or body (`body`). With Ransack it is as simple as that:

{% highlight ruby %}
Post.ransack(title_or_body_cont: 'active admin').result
{% endhighlight %}

In addition, Ransack allows you to search for records by referring to associated models. For
example, let's add search by comments (`Comment#text`):

{% highlight ruby %}
Post.ransack(comments_text_cont: 'I hate type annotations!').result
{% endhighlight %}

As you might guess, such things can grow quickly. Using complex parameters in several places can
lead to problems as well. Ransack suggests using `#ransack_alias` as a solution. Let's add filtering
by an author name to search by comment and give it a short alias: `comments` which in the future can
be used with the predicates we need:

{% highlight ruby %}
# app/models/post.rb
class Post < ActiveRecord::Base
  has_many :comments

  ransack_alias :comments, :comments_text_or_comments_author
end

Post.ransack(comments_cont: 'Matz').result
{% endhighlight %}

Now that we have learned how Ransack allows us to structure requests. Let's finally move to how we
can use it in Active Admin.

### Ransack: combined filters

Let's take the example above and use them to filter the Active Admin resource:

{% highlight ruby %}
# app/admin/posts.rb
ActiveAdmin.register Post do
  preserve_default_filters!
  filter :title_or_body_cont,
         as: :string,
         label: I18n.t('active_admin.filters.title_or_body_cont')
  filter :comments,
         as: :string
end
{% endhighlight %}

![Combined filter](/assets/an-unofficial-active-admin-guide/07-combined-filter.jpg)

Basically, that's it, very straightforward. The only thing I'd like to note is the
`#preserve_default_filters!` method which renders default filters.

### Ransack: scope-filters

By default, Ransack allows you to filter by all attributes and relationships in the model. It can
be dangerous from a security point of view, so please note that it is possible to restrict access to
certain fields and links using the `ransackable_attributes`, `ransackable_associations`, and
`ransackable_scopes` methods. I would leave authorization issues outside the scope of the article
(especially since Active Admin has a detailed section in its
[documentation](https://activeadmin.info/13-authorization-adapter.html)), so let's only pay
attention to the `ransackable_scopes` method.

Unlike other authorization methods, `ransackable_scopes` doesn't allow using any scope by default.
Thus, to be able to filter by scope (or any other method of the model class), you need to return its
name from `.ransackable_scopes`.

For example, let's add a filter by the number of comments using `scope`:

{% highlight ruby %}
# app/models/post.rb
class Post < ActiveRecord::Base
  has_many :comments

  scope :comments_count_gt, (lambda do |comments_count|
     joins(:comments)
       .group('posts.id')
       .having('count(comments.id) > ?', comments_count)
  end)

  def self.ransackable_scopes(auth_object = nil)
    [:comments_count_gt]
  end
end
{% endhighlight %}

Note `auth_object`: in theory, this is the object by which you can define an authorization strategy.
I would expect `current_user` to be passed here, but Active Admin does not do it.

We added a scope and returned its name to `.ransackable_scopes`, the only thing left is to add
a filter to the Active Admin resource:

{% highlight ruby %}
# app/admin/posts.rb
ActiveAdmin.register Post do
  filter :comments_count_gt,
         as: :number,
         label: I18n.t('active_admin.filters.comments_count_gt')
{% endhighlight %}

![Comments count filter](/assets/an-unofficial-active-admin-guide/08-comments-count-filter.jpg)

There's one little thing left: if we try to filter all the articles with two or more comments,
everything would be fine, but if we try to submit `1`, we would get an error:

![ArgumentError](/assets/an-unofficial-active-admin-guide/09-argument-error.jpg)

It is a type conversion that Ransack does for historical reasons. To disable this questionable
feature, we should add an initializer with the specified parameter `sanitize_custom_scope_booleans`:

{% highlight ruby %}
# /config/initializers/ransack.rb
Ransack.configure do |config|
  config.sanitize_custom_scope_booleans = false
end
{% endhighlight %}

There you go, now the filter works even if we submit `1` as an argument, and we know how to use
scope-based filters.

### Ransack: what's next

First of all, you should take a look at Active Admin's documentation
[regarding filters](https://activeadmin.info/3-index-pages.html#index-filters). You can continue
your overview with the official [README](https://github.com/activerecord-hackery/ransack) and
[wiki](https://github.com/activerecord-hackery/ransack/wiki), where, among other things, you can
find view-helpers to create custom search forms.

For especially complicated cases, you can consider learning how to create
[custom predicates](https://github.com/activerecord-hackery/ransack/wiki/Custom-Predicates) and
[Ransackers](https://github.com/activerecord-hackery/ransack/wiki/Using-Ransackers) - extensions
that convert parameters directly into Arel (internal library ActiveRecord, used to build SQL
queries).

## Conclusion

I hope that the article allowed you to look at Active Admin from a new perspective, and maybe even
inspired you to refactor a class or two in your projects.

I tried not to repeat the official Active Admin
[documentation](https://activeadmin.info/documentation.html), which describes many interesting
features of the library, such as authorization and the use of decorators. Therefore make sure to
check it once again.

Russian version of the article is published in the [Domclick blog](https://habr.com/ru/company/domclick/blog/514506/).
