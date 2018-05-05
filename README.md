# ajax-datatables-rails

[![GitHub license](https://img.shields.io/github/license/jbox-web/ajax-datatables-rails.svg)](https://github.com/jbox-web/ajax-datatables-rails/blob/master/LICENSE)
[![Gem](https://img.shields.io/gem/v/ajax-datatables-rails.svg)](https://rubygems.org/gems/ajax-datatables-rails)
[![Gem](https://img.shields.io/gem/dtv/ajax-datatables-rails.svg)](https://rubygems.org/gems/ajax-datatables-rails)
[![Build Status](https://travis-ci.org/jbox-web/ajax-datatables-rails.svg?branch=master)](https://travis-ci.org/jbox-web/ajax-datatables-rails)
[![Code Climate](https://codeclimate.com/github/jbox-web/ajax-datatables-rails/badges/gpa.svg)](https://codeclimate.com/github/jbox-web/ajax-datatables-rails)
[![Test Coverage](https://codeclimate.com/github/jbox-web/ajax-datatables-rails/badges/coverage.svg)](https://codeclimate.com/github/jbox-web/ajax-datatables-rails/coverage)

**Important : This gem is targeted at DataTables version 1.10.x.**

It's tested against :

* Rails 4.2.10 / 5.0.7 / 5.1.6 / 5.2.0
* Ruby 2.3.7 / 2.4.4 / 2.5.1
* Postgresql 9.6
* MySQL 5.6
* Oracle XE 11.2 (thanks to [travis-oracle](https://github.com/cbandy/travis-oracle))
* SQLite3

## Description

> [DataTables](https://datatables.net/) is a nifty jQuery plugin that adds the ability to paginate, sort, and search your html tables.
> When dealing with large tables (more than a couple of hundred rows) however, we run into performance issues.
> These can be fixed by using server-side pagination, but this breaks some DataTables functionality.
>
> `ajax-datatables-rails` is a wrapper around DataTables ajax methods that allow synchronization with server-side pagination in a Rails app.
> It was inspired by this [Railscast](http://railscasts.com/episodes/340-datatables).
> I needed to implement a similar solution in a couple projects I was working on, so I extracted a solution into a gem.
>
> Joel Quenneville (original author)
>
> I needed a good gem to manage a lot of DataTables so I chose this one :)
>
> Nicolas Rodriguez (current maintainer)

The final goal of this gem is to **generate a JSON** content that will be given to jQuery DataTables.
All the datatable customizations (header, tr, td, css classes, width, height, buttons, etc...) **must** take place in the [javascript definition](#5-wire-up-the-javascript) of the datatable.
jQuery DataTables is a very powerful tool with a lot of customizations available. Take the time to [read the doc](https://datatables.net/reference/option/).


## Warning

**Breaking changes :** the *v0.4* version is a **major break** from *v0.3*.

The core has been rewriten to remove dependency on [Kaminari](https://github.com/kaminari/kaminari) or [WillPaginate](https://github.com/mislav/will_paginate).

It also brings a new (more natural) way of defining columns, based on hash definitions (and not arrays) and add some filtering options for column search.

[See below](#3-customize-the-generated-datatables-class) for more infos.

To migrate on the v0.4 you'll need to :

* update your DataTables classes to remove all the `extend` directives
* switch to hash definitions of `view_columns`
* update your views to declare your columns bindings ([See here](#5-wire-up-the-javascript))


## Installation

Add these lines to your application's Gemfile:

```ruby
gem 'ajax-datatables-rails'
```

And then execute:

```sh
$ bundle install
```

We assume here that you have already installed [jQuery DataTables](https://datatables.net/).

You can install jQuery DataTables :

* with the [`jquery-datatables`](https://github.com/mkhairi/jquery-datatables) gem
* by adding the assets manually (in `vendor/assets`)
* with [Rails webpacker gem](https://github.com/rails/webpacker) (see [here](/doc/webpack.md) for more infos)


## Configuration

Generate the `ajax-datatables-rails` config file with this command :

```sh
$ bundle exec rails generate datatable:config
```

Doing so, will create the `config/initializers/ajax_datatables_rails.rb` file with the following content :

```ruby
AjaxDatatablesRails.configure do |config|
  # available options for db_adapter are: :pg, :mysql, :mysql2, :sqlite, :sqlite3
  # config.db_adapter = :pg

  # Or you can use your rails environment adapter if you want a generic dev and production
  # config.db_adapter = Rails.configuration.database_configuration[Rails.env]['adapter'].to_sym

  # available options for orm are: :active_record, :mongoid
  # config.orm = :active_record
end
```

Uncomment the `config.db_adapter` line and set the corresponding value to your database and gem. This is all you need.

Uncomment the `config.orm` line to set `active_record or mongoid` if included in your project. It defaults to `active_record`.

#### Note

Currently `AjaxDatatablesRails` only supports `ActiveRecord` as ORM for performing database queries.

Adding support for `Sequel`, `Mongoid` and `MongoMapper` is (more or less) a planned feature for this gem.

If you'd be interested in contributing to speed development, please [open an issue](https://github.com/antillas21/ajax-datatables-rails/issues/new) and get in touch.


## Quick start (in 5 steps)

The following examples assume that we are setting up `ajax-datatables-rails` for an index page of users from a `User` model,
and that we are using Postgresql as our db, because you **should be using it**. (It also works with other DB, see above, just be sure to have [configured the right adapter](#configuration))

The goal is to render a users table and display : `id`, `first name`, `last name`, `email`, and `bio` for each user.

Something like this:

|ID |First Name|Last Name|Email                 |Brief Bio|
|---|----------|---------|----------------------|---------|
| 1 |John      |Doe      |john.doe@example.net  |Is your default user everywhere|
| 2 |Jane      |Doe      |jane.doe@example.net  |Is John's wife|
| 3 |James     |Doe      |james.doe@example.net |Is John's brother and best friend|

Here the steps we're going through :

1. [Generate the datatable class](#1-generate-the-datatable-class)
2. [Build the View](#2-build-the-view)
3. [Customize the generated Datatables class](#3-customize-the-generated-datatables-class)
4. [Setup the Controller action](#4-setup-the-controller-action)
5. [Wire up the Javascript](#5-wire-up-the-javascript)

### 1) Generate the datatable class

Run the following command:

```sh
$ rails generate datatable User
```

This will generate a file named `user_datatable.rb` in `app/datatables`.
Open the file and customize in the functions as directed by the comments.

Take a look [here](#generator-syntax) for an explanation about the generator syntax.


### 2) Build the View

You should always start by the single source of truth, which is your html view.

* Set up an html `<table>` with a `<thead>` and `<tbody>`
* Add in your table headers if desired
* Don't add any rows to the body of the table, DataTables does this automatically
* Add a data attribute to the `<table>` tag with the url of the JSON feed, in our case is the `users_path` as we're pointing to the `UsersController#index` action


```html
<table id="users-datatable" data-source="<%= users_path(format: :json) %>">
  <thead>
    <tr>
      <th>ID</th>
      <th>First Name</th>
      <th>Last Name</th>
      <th>Email</th>
      <th>Brief Bio</th>
    </tr>
  </thead>
  <tbody>
  </tbody>
</table>
```


### 3) Customize the generated Datatables class

#### a. Declare columns mapping

First we need to declare in `view_columns` the list of the model(s) columns mapped to the data we need to present.
In this case: `id`, `first_name`, `last_name`, `email` and `bio`.

This gives us:

```ruby
def view_columns
  @view_columns ||= {
    id:         { source: "User.id" },
    first_name: { source: "User.first_name", cond: :like, searchable: true, orderable: true },
    last_name:  { source: "User.last_name",  cond: :like },
    email:      { source: "User.email" },
    bio:        { source: "User.bio" },
  }
end
```

**Notes :** by default `orderable` and `searchable` are true and `cond` is `:like`.

`cond` can be :

* `:like`, `:start_with`, `:end_with` for string or full text search
* `:eq`, `:not_eq`, `:lt`, `:gt`, `:lteq`, `:gteq`, `:in` for numeric
* `:date_range` for date range (only for Rails > 4.2.x, see [here](#daterange-search))
* `:null_value` for nil field
* `Proc` for whatever (see [here](https://github.com/ajahongir/ajax-datatables-rails-v-0-4-0-how-to/blob/master/app/datatables/city_datatable.rb) for real example)

See [here](#columns-syntax) to get more details about columns definitions and how to play with associated models.

You can customize or sanitize the search value passed to the DB by using the `:formatter` option with a lambda :

```ruby
def view_columns
  @view_columns ||= {
    id:         { source: "User.id" },
    first_name: { source: "User.first_name" },
    last_name:  { source: "User.last_name" },
    email:      { source: "User.email", formatter: -> (o) { o.upcase } },
    bio:        { source: "User.bio" },
  }
end
```

The object passed to the lambda is the search value.

#### b. Map data

Then we need to map the records retrieved by the `get_raw_records` method to the real values we want to display :

```ruby
def data
  records.map do |record|
    {
      id:         record.id,
      first_name: record.first_name,
      last_name:  record.last_name,
      email:      record.email,
      bio:        record.bio,
      DT_RowId:   record.id, # This will automagically set the id attribute on the corresponding <tr> in the datatable
    }
  end
end
```
You can either use the v0.3 Array style for your columns :

This method builds a 2d array that is used by datatables to construct the html
table. Insert the values you want on each column.

```ruby
def data
  records.map do |record|
    [
      record.id,
      record.first_name,
      record.last_name,
      record.email,
      record.bio
    ]
  end
end
```

The drawback of this method is that you can't pass the `DT_RowId` so it's tricky to set the id attribute on the corresponding `<tr>` in the datatable (need to be done on JS side).

[See here](#using-view-helpers) if you need to use view helpers like `link_to`, `mail_to`, etc...

#### c. Get Raw Records

This is where your query goes.

```ruby
def get_raw_records
  User.all
end
```

Obviously, you can construct your query as required for the use case the datatable is used.

Example:

```ruby
def get_raw_records
  User.active.with_recent_messages
end
```

You can put any logic in `get_raw_records` [based on any parameters you inject](#pass-options-to-the-datatable-class) in the `Datatable` object.

**IMPORTANT :** Because the result of this method will be chained to `ActiveRecord` methods for sorting, filtering and pagination,
make sure to return an `ActiveRecord::Relation` object.

#### d. Additional data

You can inject other key/value pairs in the rendered JSON by defining the `#additional_data` method :

```ruby
def additional_data
  {
    foo: 'bar'
  }
end
```

Very useful with https://github.com/vedmack/yadcf to provide values for dropdown filters.


### 4) Setup the Controller action

Set the controller to respond to JSON

```ruby
def index
  respond_to do |format|
    format.html
    format.json { render json: UserDatatable.new(params) }
  end
end
```

Don't forget to make sure the proper route has been added to `config/routes.rb`.

[See here](#pass-options-to-the-datatable-class) if you need to inject params in the `UserDatatable`.

### 5) Wire up the Javascript

Finally, the javascript to tie this all together. In the appropriate `coffee` file:

```coffeescript
# users.coffee

$ ->
  $('#users-datatable').dataTable
    processing: true
    serverSide: true
    ajax: $('#users-datatable').data('source')
    pagingType: 'full_numbers'
    columns: [
      {data: 'id'}
      {data: 'first_name'}
      {data: 'last_name'}
      {data: 'email'}
      {data: 'bio'}
    ]
    # pagingType is optional, if you want full pagination controls.
    # Check dataTables documentation to learn more about
    # available options.
```

or, if you're using plain javascript:

```javascript
// users.js

jQuery(document).ready(function() {
  $('#users-datatable').dataTable({
    "processing": true,
    "serverSide": true,
    "ajax": $('#users-datatable').data('source'),
    "pagingType": "full_numbers",
    "columns": [
      {"data": "id"},
      {"data": "first_name"},
      {"data": "last_name"},
      {"data": "email"},
      {"data": "bio"}
    ]
    // pagingType is optional, if you want full pagination controls.
    // Check dataTables documentation to learn more about
    // available options.
  });
});
```

## Advanced usage

### Using view helpers

Sometimes you'll need to use view helper methods like `link_to`, `mail_to`,
`edit_user_path`, `check_box_tag` and so on in the returned JSON representation returned by the [`data`](#b-map-data) method.

To have these methods available to be used, this is the way to go:

```ruby
class MyCustomDatatable < AjaxDatatablesRails::Base
  # either define them one-by-one
  def_delegator :@view, :check_box_tag
  def_delegator :@view, :link_to
  def_delegator :@view, :mail_to
  def_delegator :@view, :edit_user_path

  # or define them in one pass
  def_delegators :@view, :check_box_tag, :link_to, :mail_to, :edit_user_path

  # ... other methods (view_columns, get_raw_records...)

  # now, you'll have these methods available to be used anywhere
  def data
    records.map do |record|
      {
        id:         check_box_tag('users[]', record.id),
        first_name: link_to(record.first_name, edit_user_path(record)),
        last_name:  record.last_name,
        email:      mail_to(record.email),
        bio:        record.bio
        DT_RowId:   record.id,
      }
    end
  end
end
```

### Using view decorators

If you want to keep things tidy in the data mapping method, you could use
[Draper](https://github.com/drapergem/draper) to define column mappings like below.

Example :

```ruby
...
  def data
    records.map do |record|
      {
        id:         record.decorate.check_box,
        first_name: record.decorate.link_to,
        last_name:  record.decorate.last_name
        email:      record.decorate.email,
        bio:        record.decorate.bio
        DT_RowId:   record.id,
      }
    end
  end
...

class UserDecorator < ApplicationDecorator
  delegate :last_name, :bio

  def check_box
    h.check_box_tag 'users[]', object.id
  end

  def link_to
    h.link_to object.first_name, h.edit_user_path(object)
  end

  def email
    h.mail_to object.email
  end

  # Just an example of a complex method you can add to you decorator
  # To render it in a datatable just add a column 'dt_actions' in
  # 'view_columns' and 'data' methods and call record.decorate.dt_actions
  def dt_actions
    links = []
    links << h.link_to 'Edit',   h.edit_user_path(object) if h.policy(object).update?
    links << h.link_to 'Delete', h.user_path(object), method: :delete, remote: true if h.policy(object).destroy?
    h.safe_join(links, '')
  end
end
```

### Pass options to the datatable class

An `AjaxDatatablesRails::Base` inherited class can accept an options hash at initialization. This provides room for flexibility when required.

Example:

```ruby
# In the controller
def index
  respond_to do |format|
    format.html
    format.json { render json: UserDatatable.new(params, user: current_user, from: 1.month.ago) }
  end
end

# The datatable class
class UnrespondedMessagesDatatable < AjaxDatatablesRails::Base

  # ... other methods (view_columns, data...)

  def user
    @user ||= options[:user]
  end

  def from
    @from ||= options[:from].beginning_of_day
  end

  def to
    @to ||= Date.today.end_of_day
  end

  # We can now customize the get_raw_records method
  # with the options we've injected
  def get_raw_records
    user.messages.unresponded.where(received_at: from..to)
  end

end
```

### Columns syntax

You can mix several model in the same datatable.

Suppose we have the following models: `User`, `PurchaseOrder`,
`Purchase::LineItem` and we need to have several columns from those models
available in our datatable to search and sort by.

```ruby
# we use the ModelName.column_name notation to declare our columns

def view_columns
  @view_columns ||= {
    first_name:       'User.first_name',
    last_name:        'User.last_name',
    order_number:     'PurchaseOrder.number',
    order_created_at: 'PurchaseOrder.created_at',
    quantity:         'Purchase::LineItem.quantity',
    unit_price:       'Purchase::LineItem.unit_price',
    item_total:       'Purchase::LineItem.item_total'
  }
end
```

### Associated and nested models

The previous example has only one single model. But what about if you have
some associated nested models and in a report you want to show fields from
these tables.

Take an example that has an `Event, Course, CourseType, Allocation, Teacher,
Contact, Competency and CompetencyType` models. We want to have a datatables
report which has the following column:

```ruby
'course_types.name'
'courses.name'
'contacts.full_name'
'competency_types.name'
'events.title'
'events.event_start'
'events.event_end'
'events.status'
```

We want to sort and search on all columns of the list.
The related definition would be :

```ruby
def view_columns
  @view_columns ||= {
    course_type:     'CourseType.name',
    course_name:     'Course.name',
    contact_name:    'Contact.full_name',
    competency_type: 'CompetencyType.name',
    event_title:     'Event.title',
    event_start:     'Event.event_start',
    event_end:       'Event.event_end',
    event_status:    'Event.status',
  }
end

def get_raw_records
  Event.joins(
    { course: :course_type },
    { allocations: {
      teacher: [:contact, { competencies: :competency_type }]
    }
  }).distinct
end
```

**Some comments for the above code :**

1. In the `get_raw_records` method we have quite a complex query having one to
many and many to many associations using the joins ActiveRecord method.
The joins will generate INNER JOIN relations in the SQL query. In this case,
we do not include all event in the report if we have events which is not
associated with any model record from the relation.

2. To have all event records in the list we should use the `.includes` method,
which generate LEFT OUTER JOIN relation of the SQL query.

**IMPORTANT :**

Make sure to append `.references(:related_model)` with any
associated model. That forces the eager loading of all the associated models
by one SQL query, and the search condition for any column works fine.
Otherwise the `:recordsFiltered => filter_records(get_raw_records).count(:all)`
will generate 2 SQL queries (one for the Event model, and then another for the
associated tables). The `:recordsFiltered => filter_records(get_raw_records).count(:all)`
will use only the first one to return from the ActiveRecord::Relation object
in `get_raw_records` and you will get an error message of **Unknown column
'yourtable.yourfield' in 'where clause'** in case the search field value
is not empty.

So the query using the `.includes()` method is:

```ruby
def get_raw_records
  Event.includes(
    { course: :course_type },
    { allocations: {
      teacher: [:contact, { competencies: :competency_type }]
    }
  }).references(:course).distinct
end
```

### Default scope

See [DefaultScope is evil](https://rails-bestpractices.com/posts/2013/06/15/default_scope-is-evil/) and [#223](https://github.com/jbox-web/ajax-datatables-rails/issues/223) and [#233](https://github.com/jbox-web/ajax-datatables-rails/issues/233).

### DateRange search

This feature works with [yadcf](https://github.com/vedmack/yadcf).

To enable the date range search, for example `created_at` :

* add a 'created_at' `<th>` in your html
* declare your column in `view_columns` : `created_at: { source: 'Post.created_at', cond: :date_range, delimiter: '-yadcf_delim-' }`
* add it in `data` : `created_at: record.decorate.created_at`
* setup yadcf to make `created_at` search field a range

### Generator Syntax

Also, a class that inherits from `AjaxDatatablesRails::Base` is not tied to an
existing model, module, constant or any type of class in your Rails app.
You can pass a name to your datatable class like this:


```sh
$ rails generate datatable users
# returns a users_datatable.rb file with a UsersDatatable class

$ rails generate datatable contact_messages
# returns a contact_messages_datatable.rb file with a ContactMessagesDatatable class

$ rails generate datatable UnrespondedMessages
# returns an unresponded_messages_datatable.rb file with an UnrespondedMessagesDatatable class
```

In the end, it's up to the developer which model(s), scope(s), relationship(s)
(or else) to employ inside the datatable class to retrieve records from the
database.

## Pro Tips

### Create indices for Postgresql

In order to speed up the `ILIKE` queries that are executed when using the default configuration, you might want to consider adding some indices.
For postgresql, you are advised to use the [gin/gist index type](http://www.postgresql.org/docs/current/interactive/pgtrgm.html).
This makes it necessary to enable the postgrsql extension `pg_trgm`. Double check that you have this extension installed before trying to enable it.
A migration for enabling the extension and creating the indices could look like this:

```ruby
def change
  enable_extension :pg_trgm
  TEXT_SEARCH_ATTRIBUTES = ['your', 'attributes']
  TABLE = 'your_table'

  TEXT_SEARCH_ATTRIBUTES.each do |attr|
    reversible do |dir|
      dir.up do
        execute "CREATE INDEX #{TABLE}_#{attr}_gin ON #{TABLE} USING gin(#{attr} gin_trgm_ops)"
      end

      dir.down do
        remove_index TABLE.to_sym, name: "#{TABLE}_#{attr}_gin"
      end
    end
  end
end
```

### Speedup JSON rendering

Install [yajl-ruby](https://github.com/brianmario/yajl-ruby), basically :

```ruby
gem 'yajl-ruby', require: 'yajl'
```

then

```sh
$ bundle install
```

That's all :) ([Automatically prefer Yajl or JSON backend over Yaml, if available](https://github.com/rails/rails/commit/63bb955a99eb46e257655c93dd64e86ebbf05651))

### Use HTTP `POST` method

Use HTTP `POST` method to avoid `414 Request-URI Too Large` error. See : [#278](https://github.com/jbox-web/ajax-datatables-rails/issues/278).

You can easily define a route concern in `config/routes.rb` and reuse it when you need it :

```ruby
Rails.application.routes.draw do
  concern :with_datatable do
    post 'datatable', on: :collection
  end

  resources :posts, concerns: [:with_datatable]
  resources :users, concerns: [:with_datatable]
end
```

then in your controllers :

```ruby
# PostsController
  def index
  end

  def datatable
    render json: PostDatatable.new(view_context)
  end

# UsersController
  def index
  end

  def datatable
    render json: UserDatatable.new(view_context)
  end
```

then in your views :

```html
# posts/index.html.erb
<table id="posts-datatable" data-source="<%= datatable_posts_path(format: :json) %>">

# users/index.html.erb
<table id="users-datatable" data-source="<%= datatable_users_path(format: :json) %>">
```

then in your Coffee/JS :

```coffee
$ ->
  $('#posts-datatable').dataTable
    ajax:
      url: $('#posts-datatable').data('source')
      type: 'POST'
    # ...others options, see [here](#5-wire-up-the-javascript)

$ ->
  $('#users-datatable').dataTable
    ajax:
      url: $('#users-datatable').data('source')
      type: 'POST'
    # ...others options, see [here](#5-wire-up-the-javascript)
```

## Tutorial

You'll find a sample project [here](https://github.com/ajahongir/ajax-datatables-rails-v-0-4-0-how-to). Its real world example.

Filtering by JSONB column values : [#277](https://github.com/jbox-web/ajax-datatables-rails/issues/277)

Use [has_scope](https://github.com/plataformatec/has_scope) gem with `ajax-datatables-rails` : [#280](https://github.com/jbox-web/ajax-datatables-rails/issues/280)

## Contributing

1. Fork it
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Added some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create new Pull Request
