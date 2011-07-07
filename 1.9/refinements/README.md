# Refinements

**Author:** Shugo Maeda  
**Date:** Sometime in 2010

## What it does

Refinements are an attempt to provide, in essence, localized monkey-patching: the ability
to define a method that will only be used in a specific (module) context. If you want to
add a new method to the Array class, but have it only be available to your library (avoiding
conflicts with other packages), refinements are what you want.

That's my go at explaining it. I recommend you read [this article by Magnus Holm](http://timelessrepo.com/refinements-in-ruby)
who does an awesome job of explaining it.

However, because this readme is licensed compatibly with Magnus's article, I'll just copy and paste it here:

# Magnus Holm's "Refinements in Ruby"

At RubyConf 2010 Shugo Maeda talked about Refinements: A proposal for a new feature in Ruby which allows you to easily override methods without affecting other parts of your program:

```
module TimeExtensions
  refine Fixnum do
    def minutes; self * 60; end
  end
end

class MyApp
  using TimeExtensions

  def initialize
    p 2.minutes
  end
end

MyApp.new    # => 120
p 2.minutes  # => NoMethodError
```

Let’s have a look at the why’s and how’s of this proposal.

## The Power of Monkey Patching

Ruby allows you to both open previously defined classes and redefine any method. In addition, Ruby doesn’t treat core classes any differently from user-defined classes, so this gives you a lot of power to completely change the behaviour Ruby. This is of course a double edged sword: You can more easily change Ruby to match your thoughts (rather than changing your thoughts to match Ruby), but it also means that everyone else now needs to follow your rules.

Getting everyone to play along nicely has proven to be a challenge, and the solution has always been solved socially. As long as two core teams (let’s say Rails and DataMapper) work together, they can quite easily solve any problems, but the real issue is when you, as a user, want to use two libraries together. The libraries may work perfectly separately, but the moment you combine them you’ll get some weird behaviour. There’s not really much to can do, other than waiting for the library to be updated (or do the work yourself).

## A wild Classbox appears!

If you’ve been following the development of Ruby, you may have heard of classboxes. They were first introduced by Alexandre Bergel, Stéphane Ducasse, and Roel Wuyts in 2003 by the paper Classboxes: A Minimal Module Model Supporting Local Rebinding. It’s essentially a way to monkey patch classes and methods, but only within the context of your code and not globally. At the moment, it’s been implemented in Smalltalk (Squak), Java and .Net, but there’s also been some work at trying to apply it to Ruby.

The refinements proposal by Shugo captures the same idea as classboxes, but it behaves slightly differently in certain cases (we’ll get back to those in a minute). The differences are not big enough to justify having both refinements and classboxes, so expect this to be the only way to safely monkey patch in Ruby in the following years (if the proposal gets accepted of course).

While this is still only a proposal, Shugo has actually implemented it in Ruby 1.9 and provided a patch, so why not install it right away so you can play with it as we go through the features?

## Installing

In order to install the refinements version of Ruby, you need to grab r29837 of trunk and apply the refinements-patch. If you’re using rvm, it’s as simple as:

``` bash
$ curl -O http://stuff.judofyr.net/refinements.diff
$ rvm install ruby-head-r29837 --patch refinements.diff
$ rvm ruby-head-r29837
```

Or manually:

``` bash
$ svn checkout -q -r 29837 http://svn.ruby-lang.org/repos/ruby/trunk ruby-refinements
$ cd ruby-refinements
$ curl http://stuff.judofyr.net/refinements.diff | patch -p1
$ autoconf
$ ./configure --prefix /usr/local/ruby-refinements
$ make
$ make install
$ export PATH="/usr/local/ruby-refinements/bin:$PATH"
```

Now you should be able to run all of the examples given in this article.

## Refine, don’t redefine

Instead of redefining or defining new methods directly on classes, you’ll create refinements:

```
module JSONGenerator
  refine String do
    def to_json; inspect end
  end
  
  refine Fixnum do
    def to_json; to_s end
  end
  
  refine Array do
    def to_json
      # Refinements can see one another, so we can use String#to_json and
      # Fixnum#to_json as part of the definition of Array#to_json.
      "[" + map { |x| x.to_json } + "]"
    end
  end
end
```

If you don’t do anything other than that, you won’t notice anything at all. However, now you can choose to use this refinement at many different scopes:

```
using JSONGenerator       # For the whole file
1.to_json

module Application
  using JSONGenerator     # For this module and any nested classes and modules
                          # E.g. this also applies to Application::Controller
  
  # It works directly inside the class definition:
  2.to_json

  # And inside methods:
  def self.hello
    3.to_json
  end
  
  class Controllers
    using JSONGenerator   # For this class and any nested classes and modules
    
    def get
      using JSONGenerator # For this method only
      [1, 2, 3].to_json
    end
  end
end
```

The great thing about refinements, is that it’s technically impossible to globally leak them. They will always be restricted to the scope you specify, and there’s nothing “above” the file scope.

That’s not always true though. Refinements are also enabled in subclasses and reopened classes, even if they are located in different files.

```
class ApplicationController
  using JSONGenerator
end

# Somewhere else:
class ApplicationController
  p 123.to_json  # Still works
end

class UsersController < ApplicationController
  p 123.to_json  # Still works
end

p 123.to_json  # This doesn't work however
```

But here comes the best part: Refinements are also carried on in class_eval, module_eval and instance_eval:

```
module Expectations
  refine Object do
    def should; ... end
  end
end

def it(msg, &blk)
  # Remember that refinements can see one another:
  Expectations.module_eval(&blk)
end

it "should be awesome" do
  :refinements.level.should == :awesome
end
```

Holy Schmoly, now we’re talking! Even this works as expected:

```
class TestScope
  using Expectations
  attr_reader :msg

  def initialize(msg)
    @msg = msg
  end
end

def it(msg, &blk)
  TestScope.new(msg).instance_eval(&blk)
end
```

Refinements are also inherited, so Rails 4 could provide this module:

```
module ActiveSupport::All
  using ActiveSupport::Autoload
  using ActiveSupport::Callbacks
  # and so on ...
end
```

And because refinements are also enabled in subclasses:

```
class ApplicationController < ActionController::Base
  using ActiveSupport::All
end

class ArticlesController < ApplicationController
  def index
    @articles = Article.where("created_at > ?", 3.days.ago)
  end
end
```

You can continue developing in the exact same way as before, but now without leaking anything into the global namespace.

(This is the moment where you’re proposing to marry Shugo.)

What’s the catch?

There’s a few things you need to be aware of. First of all, there might be a little decrease in performance. Hopefully this will be resolved (or turn out to be insignificant) in the future. Other issues:

```
#include and #using are completely separated:
module Rack::Utils
  refine Object do
    def call; ... end
  end
  
  def escape_html; ... end
  end
end

# I want use both:
module Camping
  include Rack::Utils
  using Rack::Utils
end
```

Because refinements are lexically scoped, it’s also not possible to combine them with an included hook:

```
module Rack::Utils
  def self.included(mod)
    # Doesn't work as expected:
    mod.send(:using, self)
  end
end

module Camping
  include Rack::Utils
end
```

You can however use the used hook:

```
module Rack::Utils
  def self.used(mod)
    mod.send(:include, self)
  end
end

module Camping
  using Rack::Utils
end
```

Singleton methods in refinements are not included:

```
module FixnumExt
  # This has no effect:
  refine Fixnum do
    def self.thing; ... end
  end
  
  # Use this instead:
  refine Fixnum.singleton_class do
    def thing; ... end
  end
end
```
You can’t refine modules:
```
module EnumerableExt
  # Error:
  refine Enumerable do
  end
end
```
## Refinements don’t have local rebinding

Another important fact is that, unlike classboxes, refinements don’t have local rebinding. Let me show you an example:

```
class CharArray
  def initialize(str)
    @array = str.unpack("C*")  # Unpacks to integers
  end
  
  def each(&blk)
    @array.each(&blk)
  end
  
  def print_each
    each { |chr| p chr }
  end
end

test = CharArray.new("Hello World")
test.print_each   # Prints a list of integers (expected)

# A refinement which overwrites CharArray#each to return one-char strings
# instead of integers:

module CharArrayStr
  refine CharArray do
    def each
      super { |c| yield c.chr }
    end
  end
end

using CharArrayStr
test.each { |x| p x }  # Prints a list of strings
test.print_each        # Prints a list of integers?!
```

At first, it might seem counter-intuitive. Why does the last line prints a list of integers? Why isn’t the refinement enabled in that method? As you might have guessed, it’s because refinements don’t have local rebinding. This means that the refinements will only apply to the scope they are enabled. The moment you call a method outside of the scope, none of the refinements apply anymore.

The advantage of this is that you can safely override methods without thinking about breaking anything else. You simply can’t refine code in another scope. However, there’s a huge disadvantage: If there’s a “core” method (like #each above) which is used by several other methods, you can’t affect the other methods.

Local rebinding might be implemented, but in that case, refinements will be renamed to classbox (since that’s what they are).

## Current status

As I’ve mentioned earlier, there is currently a patch available that builds cleanly on top of r29837. There are some implementation details which might need to be resolved, but as far as I know, both matz and ko1 are positive for merging the patch.

## Resources

* [The slides to Shugo’s talk](http://www.slideshare.net/ShugoMaeda/rc2010-refinements)
* [The proposal at ruby-core](http://redmine.ruby-lang.org/issues/show/4085)

Refinements are very much a work in progress, so if you have any more details or questions about how they work, feel free to contact me on timeless@judofyr.net so I can keep this article up to date.

(Thanks to Shugo Maeda for explaining in detail how refinements work, and [Rune Botten](http://runerb.com/) and [Peter Aronoff](http://ithaca.arpinum.org/) for reading drafts of this.)