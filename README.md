# ActionArgs : RoR plugin

## Motivation

Controller actions are special methods in that they interface with
external programs.  In particular:

* They have no formal parameters to allow one to declare what input
  they expect.  Rather, they're provided a special hash-like object
  called 'params' comprising all supplied arguments from the client.
* And all those arguments arrive as Strings (or collections of
  same).

This means that the action has to do work to:

* Sort out what it was given.
* Determine whether it's sufficient and valid.
* Turn it into what it actually needs.

That's a lot of work for each action to have to do.

### What Is Expected?

The client-supplied arguments are not guaranteed to match the action's
expectations.  From the action's perspective, these arguments may fall
in one of three varieties:

* Required. Necessary for the action's proper functioning.
* Optional.  Their mere presence or absence may impact the action's
  behavior in addition to their value.
* Unused.  From the action's perspective, they're spurious.  (They
  may be used only for logging, for example.)

The action must deal with these different varieties of arguments
differently.

### Writing Actions

When writing controller actions, dealing with Unused arguments is
usually easy (ignore 'em), but with an important caveat: one must be
careful not to pass along the params hash intact to other code, as the
presence of spurious key-value pairs may induce improper behavior.

Dealing with Required and Optional arguments, however, is trickier.

* For Required args, one must explicitly check for their presence
  before using them, or exception-handle if absent.
* For Optional args, one must:
  * likewise check for their presence, and either
    * possibly branch on their presence or absence, or
    * supply a default value if absent.
* For both Required and Optional args, one
  * _must_ (usually) convert the Strings into values of the type one
    really needs (such as ints or booleans or what-have-you).
  * _may_ need to validate those values before using them.

So writing good, robust actions is difficult work.

### Reading Actions

Controller code is not only difficult to write, it can also be
difficult to understand.

First, this is because so much of the action code may be dedicated to
digesting the parameters that the (other) "real" work of the
controller is obscured.

Second, the action's expectations about arguments are not made
immediately apparent.  Sometimes, an argument is expect from the
client but not used directly in the controller and instead passed
along to another object (or chain of them) which may ignore it, or
convert it to another type, or otherwise modify its value, and then
perhaps validate it before using it.  Considerable detective work may
be necessary to determine which arguments are actually used and how.

### ActionArgs to the Rescue

The goal of ActionArgs is to solve these problems.  It's a
work-in-progress, but already it should prove useful for mitigating
some of these problems.

## How to Use It

Or, "Pulling Args from Params" (http://www.youtube.com/watch?v=3WngGeI9lnA).

### Plugin

First, include the plugin:

```
# app/controllers/application.rb
class ApplicationController < ActionController::Base
  include ActionController::ArgyBargy
end
````

### Declare Your Args

Then, in your controller, you may:

* declare for each action:
  * which arguments you expect, and
  * some info about each
* access them via {{{params}}}' evil twin, {{{args}}}


Here's an excessively-commented example:

```
# app/controllers/bojacks_controller.rb
class BojacksController < ApplicationController

  VERTICALS = [:books, :games, :other_crap]
    
  args_for :my_action do
    # Required arg called :vertical.  (If absent, raises ActionArgs::ArgumentError.)
    # Value is a Symbol (converted from supplied String).
    # Ensure value's validity or raise ActionArgs::ArgumentError.
    req(:vertical).as(:symbol).validate {|s| VERTICALS.include? s }

    # Optional string arg called :filter.
    # If provided, its value is downcased automatically.
    # If not provided, its value is nil.
    opt(:filter).as(:string).munge(:downcase)

    # Optional hash called :paging.
    # If not provided, :paging is nil.
    #
    # One may not supply a default value for an entire (optional) hash.
    # Instead:
    #  - make the hash required, but
    #  - make each of its k:v pairs optional (with defaults).
    #
    # (Here, however, the hash is optional, but if present,
    # its members are required.)
    opt_hash(:paging) do
      # If :paging hash is present...
    
      # Required int (Fixnum, really) called :offset.
      # Must be non-negative (or raises ActionArgs::ArgumentError).
      req(:offset).as(:int).validate {|i| i >= 0 }
    
      # Required int called :limit.
      # Must be positive (or raises ActionArgs::ArgumentError).
      req(:limit).as(:int). validate {|i| i > 0  }
    end
    
    # Optional boolean arg called :show_related_p.
    # If absent, default value is false.
    opt(:show_related_p).as(:bool).default(false)
  end

  def my_action
    # you can use...
    args[:vertical]        # a Symbol, one of VERTICALS
    args[:filter]          # nil -or- a downcased String
    args[:paging]          # nil -or- {:offset => <non-neg. int>, :limit => <pos. int>}
    args[:show_related_p]  # true -or- false
  end

end
```

## Exceptions

But wait, what if something goes wrong?  What if a declaration makes no sense?

### ActionArgs::ConfigError

If there's an error in an args_for() method -- rather, an error which
is detectable by the library --, then your app server will never start
up.  ActionArgs will raise an exception of type ConfigError,
explaining what it thinks you did wrong.  Some bojacked examples:

```
args_for :action1
  req(:foo).as(:int)
  opt(:foo).as(:int)                # repeated arg.
end
    
args_for :action2
  req(:foo).default('bar')          # required args cannot have defaults.
end
    
args_for :action3
  opt(:id).as(:int).default(true)   # default of wrong type.
end
    
args_for :action4
  # Default does not validate.  (Or really, the validate method is janky.)
  opt(:vertical).as(:symbol).default(:books).
    validate {|sym| ['books', 'games'].include? sym }
end
````

So that's good.


### ActionArgs::ArgumentError

If your declaration seems sound, but the supplied parameters are not,
then the library will raise an exception of type
ActionArgs::ArgumentError.  How to handle this?  There are two
options:

* By default, the plugin will handle the exception for you in a
  standard way, using ApplicationController#rescue_action_locally.
  Your controller action code will never get called.  If that's good
  by you, you're golden.
* If you want your controller action to get called regardless of
  ActionArgs errors, then you'll need to add to your
  args_for() declaration, like this:

```
args_for :my_action, :raise_p => false do
  ...
end
```

If you tell #args_for not to raise, then your action code will be
called, and you may ask of the args object what happened by inspecting
the exceptions it gathered up (args.errors -- yes, yes, it really should
be named args.exceptions instead):

```
def my_action
  if !args.valid?
    # action-specific exception-handling code...
    errors_str = args.errors.map(&:to_s).join("\n")
    render_json(:success   => false,
                :exception => "Some bojackedness occurred: #{errors_str}")
  else
    # "real" action code...
  end
end
```

And that's it.

