# A Tiny Event System for Bash

This is a simple event listener/callback API for bash programs.  Events can be one-time or repeated, listeners can be added or removed, and any valid identifier can be an event.  Bash 4 is **not** required: this code should work just fine with OS X's old bash 3.2.  No external programs are used, so it's fast and portable.

**Contents**

<!-- toc -->

- [Installation, Requirements And Use](#installation-requirements-and-use)
- [Events API](#events-api)
  * [Passing Arguments](#passing-arguments)
- [Utilities](#utilities)
  * [event valid](#eventvalid)
  * [event quote](#eventquote)
- [License](#license)

<!-- tocstop -->

### Installation, Requirements And Use

Copy and paste the [code](bashup.events) into your script, or place it on `PATH` and `source bashup.events`.  (If you have [basher](https://github.com/basherpm/basher), you can `basher install bashup/events` to get it installed on your `PATH`.)  The code is licensed [CC0](http://creativecommons.org/publicdomain/zero/1.0/), so you are not required to add any attribution or copyright notices to your project.

````sh
    $ source bashup.events
````

### Events API

Events can be named using any sequence of alphanumeric or `_` characters.  Invoking any API command (other than `event valid`) with an invalid event name will abort program execution with an exit code of 64 (EX_USAGE), and a message on stderr.

The main API commands are:

* `event on` *event cmd [args...]* -- subscribes *cmd args...* as a callback to *event*, if it's not already added
* `event off` *event cmd [args...]* -- unsubscribes the *cmd args...* callback from *event*
* `event has` *event [cmd [args...]]* -- returns truth if *event* has a registered callback of *cmd args...*, or if no *cmd...* is given, returns truth if *any* callbacks have been registered for *event*.
* `event emit` *event data...* -- emit a "repeatable" event, by invoking all the callbacks for *event*, passing *data...* as additional arguments to each callback.  Callbacks added to the event while this function is running will not take effect until a subsequent `send` or `drain` of the event, and existing callbacks remain subscribed.
* `event fire` *event data...* -- fire a "one shot" event, by invoking all the callbacks for *event*, passing *data...* as additional arugments to each  callback.  All callbacks are removed from the event, and new callbacks added during the drain will be invoked as soon as all previously-added callbacks have been invoked.



Using these functions, you can implement both repeatable and one-shot events, e.g.:

````sh
# Subscribe to events using event on

    $ event on event1 echo "got event1"
    $ event on event1 echo "is this cool or what?"

# Fire events with event emit

    $ event emit event1
    got event1
    is this cool or what?

# Unsubscribe using event off:

    $ event off event1 echo "got event1"
    $ event emit event1
    is this cool or what?

# Test susbscription with event has

    $ event has event1 echo "is this cool or what?" && echo "cool!"
    cool!
    $ event has event1 echo "got event1" || echo "nope!"
    nope!

# `event has` with no callback tests for any callbacks at all

    $ event has event1 && echo "yes, there are some callbacks"
    yes, there are some callbacks

    $ event has something_else || echo "but not for this other event"
    but not for this other event

# event fire removes callbacks and handles nesting:

    $ event on event1 event on event1 echo "nested!"
    $ event fire event1
    is this cool or what?
    nested!

    $ event emit event1   # all callbacks gone now

````

#### Passing Arguments

When emitting or firing an event, you can pass additional arguments that will be added to the end of the arguments supplied to the given callbacks.  The callbacks, however, will only receive these arguments if they were registered to do so, by adding a `/` at the end of the event name, followed by the maximum number of arguments the callback is prepared to receive:

````sh
# Callbacks can receive extra arguments sent by emit/fire:

    $ event on "event2"/2 echo "Args:"  # accept up to 2 arguments
    $ event fire event2 foo bar baz
    Args: foo bar

````

The reason an argument count is required, is because one purpose of  an event system is to be *extensible*.  If an event adds new arguments over time, old callbacks may break if they weren't written in such a way as to ignore the new arguments.  Requiring an explicit request for arguments avoids this problem.

If the nature of the event is that it emits a *variable* number of arguments, however, you can register your callback with `/_`, which means "receive *all* the arguments, no matter how many".  You should only use it in places where you can definitely handle any number of arguments, or else you may run into unexpected behavior.

````sh
# Why variable arguments lists aren't the default:
    $ event on "cleanup"/_ echo "rm -rf"
    $ event emit "cleanup" foo
    rm -rf foo

# New release...  "cleanup" event adds a new argument
    $ event emit "cleanup" foo /
    rm -rf foo /

````

`event on`, `event off`, and `event has` all accept argument counts when adding, removing, or checking for callbacks.  Callbacks with different argument counts are considered to be *different* callbacks:

````sh
# Only one argument:

    $ event on "myevent"/1 echo
    $ event emit myevent foo bar baz
    foo

# Different count = different callbacks:

    $ event has "myevent"/1 echo && echo got it
    got it
    $ event has "myevent"/2 echo || echo nope
    nope

# Add 2 argument version:

    $ event on "myevent/2" echo
    $ event emit myevent foo bar baz
    foo
    foo bar

# Remove the 2-arg version, add unlimited version:

    $ event off "myevent"/2 echo
    $ event on "myevent"/_ echo

    $ event emit myevent foo bar baz
    foo
    foo bar baz

# Unlimited version is distinct, too:

    $ event has "myevent"/_ echo && echo got it
    got it
    $ event has "myevent"/2 echo || echo nope
    nope

# As is the zero-arg version:

    $ event has "myevent" echo || echo nope
    nope

# But the zero-arg version can be implicit or explicit:

    $ event on  "myevent" echo
    $ event has "myevent" echo && echo got it
    got it
    $ event has "myevent"/0 echo && echo got it
    got it

    $ event off "myevent"/0 echo
    $ event has "myevent" echo || echo nope
    nope
````

### Utilities

#### event valid

`event valid` *string* returns truth if *string* is safe to use as an event name (without an argument count).

````sh
    $ event valid foo && echo yep
    yep
    $ event valid foo.bar || echo nope
    nope
````

#### event quote

`event quote` *args* sets `$REPLY` to a space-separated list of the given arguments in shell-quoted form (using `printf %q`).  The resulting string is safe to `eval`, in the sense that the arguments are guaranteed to expand to the same values (and number of values) as were originally given.

````sh
    $ event quote a b c && echo "$REPLY"
    a b c
    $ event quote "a b c" && echo "$REPLY"
    a\ b\ c
    $ event quote x "" y && echo "$REPLY"
    x '' y
    $ event quote && echo "'$REPLY'"
    ''
````



### License

<p xmlns:dct="http://purl.org/dc/terms/" xmlns:vcard="http://www.w3.org/2001/vcard-rdf/3.0#">
  <a rel="license" href="http://creativecommons.org/publicdomain/zero/1.0/"><img src="https://licensebuttons.net/p/zero/1.0/80x15.png" style="border-style: none;" alt="CC0" /></a><br />
  To the extent possible under law, <a rel="dct:publisher" href="https://github.com/pjeby"><span property="dct:title">PJ Eby</span></a>
  has waived all copyright and related or neighboring rights to <span property="dct:title">bashup/events</span>.
This work is published from: <span property="vcard:Country" datatype="dct:ISO3166" content="US" about="https://github.com/bashup/events">United States</span>.
</p>