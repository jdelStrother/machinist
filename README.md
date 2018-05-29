# Machinist 3

*Fixtures aren't fun. Machinist was.*

[![Gem version](https://badge.fury.io/rb/machinist_redux.svg)](https://rubygems.org/gems/machinist_redux)
[![Gem downloads](https://img.shields.io/gem/dt/machinist_redux.svg?style=flat-square)](https://rubygems.org/gems/machinist_redux)
[![Build Status](https://travis-ci.org/dominicsayers/machinist.svg?branch=master)](https://travis-ci.org/dominicsayers/machinist)
[![Code Climate](https://codeclimate.com/github/dominicsayers/machinist/badges/gpa.svg)](https://codeclimate.com/github/dominicsayers/machinist)
[![Test Coverage](https://codeclimate.com/github/dominicsayers/machinist/badges/coverage.svg)](https://codeclimate.com/github/dominicsayers/machinist/coverage)
[![Dependencies](https://badges.depfu.com/badges/f5f7b9d12baad9fbe4070964edfc75d6/overview.svg)](https://depfu.com/github/dominicsayers/machinist)
[![Security](https://hakiri.io/github/dominicsayers/machinist/master.svg)](https://hakiri.io/github/dominicsayers/machinist/master)

- [Home page](https://github.com/dominicsayers/machinist)
- [Google group](https://groups.google.com/group/machinist-users), for support
- [Bug tracker](https://github.com/dominicsayers/machinist/issues), for reporting Machinist bugs

If you want Machinist 1, [go here](https://github.com/notahat/machinist/tree/1.0-maintenance).

If you want support for Rails 3 or Rubies prior to 2.2, [go here](https://github.com/attilagyorffy/machinist).

## Status

This is a fork of [Pete Yandell's Machinist](http://github.com/notahat/machinist). The original gem was abandoned for the reasons given below, as are most of its forks. The purpose of this fork is to keep Machinist under maintenance for legacy projects that have upgraded to Ruby 2.2 or later and Rails 4.2 or later, but still have Machinist factories in the test environment.

I'm pleased to say that Pete's code runs fine under Ruby 2.2, 2.3, 2.4 & 2.5 and Rails 4.2, 5.0, 5.1 and 5.2. I have allowed [RuboCop](https://github.com/bbatsov/rubocop) and [RuboCop RSpec](https://github.com/backus/rubocop-rspec) to suggest some changes, and I have converted to the more up-to-date RSpec syntax using [Transpec](https://github.com/yujinakayama/transpec). Few if any manual changes were needed.

Pete Yandell's reason for abandoning Machinist is that he found himself with less and less need for factories in tests. He recommends Bo Jeanes' [excellent article on the topic](http://bjeanes.com/2012/02/factories-breed-complexity).

## Introduction

Machinist makes it easy to create objects for use in tests. It generates data
for the attributes you don't care about, and constructs any necessary
associated objects, leaving you to specify only the fields you care about in
your test. For example:

```ruby
describe Comment, 'without_spam scope' do
  it "doesn't include spam" do
    # This will make a Comment, a Post, and a User (the author of the
    # Post), generate values for all their attributes, and save them:
    spam = Comment.make!(spam: true)

    Comment.without_spam.should_not include(spam)
  end
end
```

You tell Machinist how to do this with blueprints:

```ruby
require 'machinist/active_record'

User.blueprint do
  username { "user#{sn}" }  # Each user gets a unique serial number.
end

Post.blueprint do
  author
  title  { "Post #{sn}" }
  body   { 'Lorem ipsum...' }
end

Comment.blueprint do
  post
  email { "commenter#{sn}@example.com" }
  body  { 'Lorem ipsum...' }
end
```

## Installation

### Upgrading from Machinist 1

See [the wiki](http://wiki.github.com/notahat/machinist/machinist-2).

### Rails 4 and 5

In your app's `Gemfile`, in the `group :test` section, add:

```ruby
gem 'machinist_redux'
```

Then run:

```ruby
bundle install
rails generate machinist:install
```

If you want Machinist to automatically add a blueprint to your blueprints file
whenever you generate a model, add the following to your `config/application.rb` inside the Application class:

```ruby
config.generators do |g|
  g.fixture_replacement :machinist
end
```

## Usage

### Blueprints

A blueprint describes how to generate an object. The blueprint takes care of
providing attributes that your test doesn't care about, leaving you to focus on
just the attributes that are important for the test.

A simple blueprint might look like this:

```ruby
Post.blueprint do
  title  { 'A Post' }
  body   { 'Lorem ipsum...' }
end
```

You can then construct a Post from this blueprint with:

```ruby
Post.make!
```

When you call `make!`, Machinist calls `Post.new`, then runs through the
attributes in your blueprint, calling the block for each attribute to generate
a value. It then saves and reloads the Post. (It throws an exception if the
Post can't be saved.)

You can override values defined in the blueprint by passing a hash to make:

```ruby
Post.make!(title: 'A Specific Title')
```

If you want to generate an object without saving it to the database, replace
`make!` with `make`.

### Unique Attributes

For attributes that need to be unique, you can call the `sn` method from
within the attribute block to get a unique serial number for the object.

```ruby
User.blueprint do
  username { "user-#{sn}" }
end
```

### Associations

If your object needs associated objects, you can generate them like this:

```ruby
Comment.blueprint do
  post { Post.make }
end
```

Calling `Comment.make!` will construct a Comment and its associated Post, and
save both.

Machinist is smart enough to look at the association and work out what sort of
object it needs to create, so you can shorten the above blueprint to:

```ruby
Comment.blueprint do
  post
end
```

If you want to override the value for post when constructing the comment, you
can do this:

```ruby
post = Post.make(title: 'A particular title')
comment = Comment.make(post: post)
```

For `has_many` and `has_and_belongs_to_many` associations, you can create
multiple associated objects like this:

```ruby
Post.blueprint do
  comments(3)  # Makes 3 comments.
end
```

### Named Blueprints

Named blueprints let you define variations on an object. For example, suppose
some of your Users are administrators:

```ruby
User.blueprint do
  name  { "User #{sn}" }
  email { "user-#{sn}@example.com" }
end

User.blueprint(:admin) do
  name  { "Admin User #{sn}" }
  admin { true }
end
```

Calling:

```ruby
User.make!(:admin)
```

will use the `:admin` blueprint.

Named blueprints call the default blueprint to set any attributes not
specifically provided, so in this example the `email` attribute will still be
generated even for an admin user.

You must define a default blueprint for any class that has a named blueprint,
even if the default blueprint is empty.

### Blueprints on Plain Old Ruby Objects

Machinist also works with plain old Ruby objects. Let's say you have a class like:

```ruby
class Post
  extend Machinist::Machinable

  attr_accessor :title
  attr_accessor :body
end
```

You can blueprint the Post class just like anything else:

```ruby
Post.blueprint do
  title { 'A title!' }
  body  { 'A body!' }
end
```

And `Post.make` will construct a new Post.

### Other Tricks

You can refer to already assigned attributes when constructing a new attribute:

```ruby
Post.blueprint do
  author { "Author #{sn}" }
  body   { "Post by #{object.author}" }
end
```

### More Details

Read the code! No, really. I wrote this code to be read.

Check out [the specs](https://github.com/dominicsayers/machinist/tree/master/spec), starting with [the spec for Machinable](https://github.com/dominicsayers/machinist/blob/master/spec/machinable_spec.rb).

## Developing

The Machinist specs and source code were written to be read, and I'm pretty
happy with them. Don't be afraid to have a look under the hood!

If you want to submit a patch:

- Fork the project.
- Make your feature addition or bug fix.
- Add tests for it. This is important so I don't break it in a
  future version unintentionally.
- Commit, do not mess with rakefile, version, or history.
  (if you want to have your own version, that is fine but bump version in a commit by itself I can ignore when I pull)
- Send me a pull request. Bonus points for topic branches.

## Contributors

Machinist was maintained by Pete Yandell ([pete@notahat.com](mailto:pete@notahat.com), [@notahat](http://twitter.com/notahat))

Other contributors include:

[Marcos Arias](http://github.com/yizzreel),
[Jack Dempsey](http://github.com/jackdempsey),
[Jeremy Durham](http://github.com/jeremydurham),
[Clinton Forbes](http://github.com/clinton),
[Perryn Fowler](http://github.com/perryn),
[Niels Ganser](http://github.com/Nielsomat),
[Jeremy Grant](http://github.com/jeremygrant),
[Jon Guymon](http://github.com/gnarg),
[James Healy](http://github.com/yob),
[Ben Hoskings](http://github.com/benhoskings),
[Evan David Light](http://github.com/elight),
[Chris Lloyd](http://github.com/chrislloyd),
[Adam Meehan](http://github.com/adzap),
[Kyle Neath](http://github.com/kneath),
[Lawrence Pit](http://github.com/lawrencepit),
[Xavier Shay](http://github.com/xaviershay),
[T.J. Sheehy](http://github.com/tjsheehy),
[Roland Swingler](http://github.com/knaveofdiamonds),
[Gareth Townsend](http://github.com/quamen),
[Matt Wastrodowski](http://github.com/towski),
[Ian White](http://github.com/ianwhite),
[Dominic Sayers](https://github.com/dominicsayers)

Thanks to Thoughtbot's [Factory
Girl](http://github.com/thoughtbot/factory_girl/tree/master). Machinist was
written because I loved the idea behind Factory Girl, but I thought the
philosophy wasn't quite right, and I hated the syntax.
