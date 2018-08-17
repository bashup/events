# Practical Event Listeners for Bash

`bashup.events` is an event listener/callback API for creating extensible bash programs.  It's small (<2k), fast (~10k events/second), and highly portable (no bash4-isms or external programs used).  Events can be one-time or repeated, listeners can be added or removed, and any string can be an event name.  (You can even have "[promises](#promise-like-events)", of a sort!)  Callbacks can be any command or function plus any number of arguments, and can even opt to receive [additional arguments supplied by the event](#passing-arguments-to-callbacks).

Other features include:

* Running a callback each time something happens ([`event on`](#event-on), [`event emit`](#event-emit))
* Running a callback the *next* time something happens, but not after that ([`event once`](#event-once))
* Alerting subscribers of an event once, making them re-subscribe for future occurrences ([`event fire`](#event-fire) )
* Alerting subscribers of a one-time only event or calculation... not just current subscribers, but *future* ones as well ([`event resolve`](#event-resolve))
* Allowing subscribed callbacks to veto a process (e.g. validation rules), using [`event all`](#event-all)
* Searching for the first callback that can successfully handle something, using [`event any`](#event-any)


#### Contents

<!-- toc -->

- [Installation, Requirements And Use](#installation-requirements-and-use)
- [Basic Operations](#basic-operations)
  * [event on](#event-on)
  * [event emit](#event-emit)
  * [event off](#event-off)
  * [event has](#event-has)
  * [event fire](#event-fire)
- [Passing Arguments To Callbacks](#passing-arguments-to-callbacks)
- [Promise-Like Events](#promise-like-events)
  * [event resolve](#event-resolve)
  * [event resolved](#event-resolved)
- [Conditional Operations](#conditional-operations)
  * [event all](#event-all)
  * [event any](#event-any)
- [Other Operations](#other-operations)
  * [event once](#event-once)
  * [event encode](#event-encode)
  * [event decode](#event-decode)
  * [event list](#event-list)
  * [event quote](#event-quote)
  * [event error](#event-error)
- [License](#license)

<!-- tocstop -->

### Installation, Requirements And Use

Copy and paste the [code](bashup.events) into your script, or place it on `PATH` and `source bashup.events`.  (If you have [basher](https://github.com/basherpm/basher), you can `basher install bashup/events` to get it installed on your `PATH`.)  The code is licensed [CC0](http://creativecommons.org/publicdomain/zero/1.0/), so you are not required to add any attribution or copyright notices to your project.

````sh
    $ source bashup.events
````

### Basic Operations

Sourcing `bashup.events` exposes one public function, `event`, that provides a variety of subcommands.  All of the primary subcommands take an event name as their first argument.

Event names can be any string, but performance is best if you limit them to pure ASCII alphanumeric or `_` characters, as all other characters have to be encoded at the start of each event command.  (And the larger the character set used, the slower the encoding process becomes.)

#### event on

`event on` *event cmd [args...]* subscribes *cmd args...* as a callback to *event*, if it's not already added:

````sh
    $ event on "event1" echo "got event1"
    $ event on "event1" echo "is this cool or what?"
````

#### event emit

`event emit` *event data...* invokes all the callbacks for *event*, passing *data...* as additional arguments to any callbacks that [registered to receive them](#passing-arguments-to-callbacks).  Callbacks added to the event while the `emit` is occurring will **not** be invoked until a subsequent occurrence of the event, and the already-added callbacks will remain subscribed (unless they unsubscribe themselves, or were registered with [`event once`](#event-once)).

````sh
    $ event emit "event1"
    got event1
    is this cool or what?
````

#### event off

`event off` *event cmd [args...]* unsubscribes the *cmd args...* callback from *event*

````sh
    $ event off "event1" echo "got event1"
    $ event emit "event1"
    is this cool or what?
````

#### event has

* `event has` *event* returns truth if *event* has any registered callbacks.
* `event has` *event cmd [args...]* returns truth if *cmd args...* has been registered as a callback for *event*.

````sh
# `event has` with no callback tests for any callbacks at all

    $ event has "event1" && echo "yes, there are some callbacks"
    yes, there are some callbacks

    $ event has "something_else" || echo "but not for this other event"
    but not for this other event

# Test for specific callback susbscription:

    $ event has "event1" echo "is this cool or what?" && echo "cool!"
    cool!
    $ event has "event1" echo "got event1" || echo "nope!"
    nope!

````

#### event fire

`event fire` *event data...* fires a "one shot" event, by invoking all the callbacks for *event*, passing *data...* as additional arguments to any callbacks that [registered to receive them](#passing-arguments-to-callbacks).  All callbacks are removed from the event, and any new callbacks added during the firing will be invoked as soon as all the previously-added callbacks have been invoked (and then are also removed from the event).

The overall idea is somewhat similar to the Javascript "promise" resolution algorithm, except that you can `fire` an event more than once, and there is no "memory" of the arguments.  (See [`event resolve`](#event-resolve) if you want something closer to a JS Promise.)

````sh
# `event fire` removes callbacks and handles nesting:

    $ mycallback() { event on event1 echo "nested!"; }
    $ event on "event1" mycallback

    $ event fire "event1"
    is this cool or what?
    nested!

    $ event emit "event1"   # all callbacks gone now

````

### Passing Arguments To Callbacks

When invoking an event, you can pass additional arguments that will be added to the end of the arguments supplied to the given callbacks.  The callbacks, however, will only receive these arguments if they were registered to do so, by adding an extra argument after the event name: an  `@` followed by the maximum number of arguments the callback is prepared to receive:

````sh
# Callbacks can receive extra arguments sent by emit/fire/resolve/all/any:

    $ event on   "event2" @2 echo "Args:"  # accept up to 2 arguments
    $ event fire "event2" foo bar baz
    Args: foo bar

````

The reason an argument count is required, is because one purpose of an event system is to be *extensible*.  If an event adds new arguments over time, old callbacks may break if they weren't written in such a way as to ignore the new arguments.  Requiring an explicit request for arguments avoids this problem.

If the nature of the event is that it emits a *variable* number of arguments, however, you can register your callback with `@_`, which means "receive *all* the arguments, no matter how many".  You should only use it in places where you can definitely handle any number of arguments, or else you may run into unexpected behavior.

````sh
# Why variable arguments lists aren't the default:

    $ event on   "cleanup" @_ echo "rm -rf"
    $ event emit "cleanup" foo
    rm -rf foo

    $ event emit "cleanup" foo /   # New release...  "cleanup" event added a new argument!
    rm -rf foo /

````

[`event on`](#event-on), [`event once`](#event-once), [`event off`](#event-off), and [`event has`](#event-has) all accept argument count specifiers when adding, removing, or checking for callbacks.  Callbacks with different argument counts are considered to be *different* callbacks:

````sh
# Only one argument:

    $ event on   "myevent" @1 echo
    $ event emit "myevent" foo bar baz
    foo

# Different count = different callbacks:

    $ event has "myevent" @1 echo && echo got it
    got it
    $ event has "myevent" @2 echo || echo nope
    nope

# Add 2 argument version (numeric value is what's used):

    $ event on   "myevent" @02 echo
    $ event emit "myevent" foo bar baz
    foo
    foo bar

# Remove the 2-arg version, add unlimited version:

    $ event off "myevent" @2 echo
    $ event on  "myevent" @_ echo

    $ event emit "myevent" foo bar baz
    foo
    foo bar baz

# Unlimited version is distinct, too:

    $ event has "myevent" @_ echo && echo got it
    got it
    $ event has "myevent" @2 echo || echo nope
    nope

# As is the zero-arg version:

    $ event has "myevent" echo || echo nope
    nope

# But the zero-arg version can be implicit or explicit, w/or without leading zeros:

    $ event on  "myevent" echo
    $ event has "myevent" echo && echo got it
    got it
    $ event has "myevent" @0 echo && echo got it
    got it

    $ event off "myevent" @00 echo
    $ event has "myevent" echo || echo nope
    nope
````

### Promise-Like Events

#### event resolve

If you have a truly one-time event, but subscribers could "miss it" by subscribing too late, you can use `event resolve` to "permanently [`fire`](#event-fire)" an event with a specific set of arguments.  Once this is done, all future `event on` calls for that event will invoke the callback *immediately* with the previously-given arguments.

There is no way to "unresolve" a resolved event within the current shell.  Trying to `resolve`, `emit`, `fire`, `any` or `all` an already-resolved event will result in an error message and a failure return of 70 (`EX_SOFTWARE`).

````sh
# Subscribers before the resolve will be fired upon resolve:

    $ event on "promised" event on "promised" @1 echo "Nested:"
    $ event on "promised" @1 echo "Plain:"

    $ event resolve "promised" value
    Plain: value
    Nested: value

# Subscribers after the resolve are fired immediately:

    $ event on "promised" event on "promised" @1 echo "Nested:"
    Nested: value

    $ event on "promised" @1 echo "Plain:"
    Plain: value

# And a resolved event never "has" any subscribers:

    $ event has "promised" || echo nope
    nope

````

#### event resolved

`event resolved` *event* returns truth if `event resolve` *event* has been called.

````sh
    $ event resolved "promised" && echo "yep"
    yep

    $ event resolved "another_promise" || echo "not yet"
    not yet
````

### Conditional Operations

#### event all

`event all` *event data*... works like [`event emit`](#event-emit), except that execution stops after the first callback that returns false (i.e., a non-zero exit code), and that exit code is returned.  Truth is returned if all events return truth.

````sh
# Use an event to validate a password

    $ validate() { echo "validating: $1"; [[ $3 =~ $2 ]]; }

    $ event on "password_check" @1 validate "has a number" '[0-9]+'
    $ event on "password_check" @1 validate "is 8+ chars" ........
    $ event on "password_check" @1 validate "has uppercase" '[A-Z]'
    $ event on "password_check" @1 validate "has lowercase" '[a-z]'

    $ event all "password_check" 'foo27' || echo "fail!"
    validating: has a number
    validating: is 8+ chars
    fail!

    $ event all "password_check" 'Blue42Schmoo' && echo "pass!"
    validating: has a number
    validating: is 8+ chars
    validating: has uppercase
    validating: has lowercase
    pass!

````

#### event any

`event any` *event data...* also works like [`event emit`](#event-emit), except that execution stops on the first callback to return truth (i.e. a zero exit code).  An exit code of 1 is returned if all events return non-zero exit codes.

````sh
    $ match() { echo "checking for $1"; REPLY=$2; [[ $1 == $3 ]]; }

    $ event on "lookup" @1 match a "got one!"
    $ event on "lookup" @1 match b "number two"
    $ event on "lookup" @1 match c "third time's the charm"

    $ event any "lookup" b && echo "match: $REPLY"
    checking for a
    checking for b
    match: number two

    $ event any "lookup" q || echo "fail!"
    checking for a
    checking for b
    checking for c
    fail!

````

### Other Operations

#### event once

`event once` *event cmd [args...]* is like [`event on`](#event-on), except that the callback is unsubscribed before it's invoked, ensuring it will be called at most once, even if *event* is emitted multiple times in a row:

````sh
    $ event once "something" @_ echo
    $ event emit "something" this that
    this that
    $ event emit "something" more stuff
````

(Note: a callback added by `event once` cannot be removed by `event off`; if you need to be able to remove such a callback you should use `event on` instead and make the callback remove itself with `event off`.)

#### event encode

`event encode` *string* sets `$REPLY` to an encoded version of *string* that is safe to use as part of a bash variable name (i.e. ascii alphanumerics and `_`).  Underscores and all other non-alphanumerics are encoded as an underscore and two hex digits.

````sh
    $ event encode "foo"     && echo $REPLY
    foo
    $ event encode "foo.bar" && echo $REPLY
    foo_2ebar
    $ event encode "foo_bar" && echo $REPLY
    foo_5fbar
````

For performance reasons, the function that handles event encoding is JITted.  Every time new non-ASCII or non-alphanumeric characters are seen, the function is rewritten to efficiently handle encoding them.  This makes encoding extremely fast when a program only ever uses a handful of punctuation characters in event names or strings passed to `event encode`.  Encoding arbitrary strings (or using them as event names) is not recommended, however, since this will "train" the encoder to run more slowly for *all* `event` operations from then on.

#### event decode

`event decode` *string* sets `$REPLY` to the original event name for *string*, turning the encoded characters back to their original values.  If multiple arguments are given, `REPLY` is an array of results.

````sh
    $ event decode "foo_2ebar_2dbaz" && echo $REPLY
    foo.bar-baz

    $ event decode "_2fspim" "_2bspam" && printf '%s\n' "${REPLY[@]}"
    /spim
    +spam
````

#### event list

`event list` *prefix* sets `REPLY` to an array of currently-defined event names beginning with *prefix*.  events that currently have listeners are returned, as are resolved events.

````sh
# event1 and event2 no longer have subscribers:

    $ event list "event" && echo "${#REPLY[@]}"
    0

# But there are some events starting with "p"

    $ event list "p" && printf '%s\n' "${REPLY[@]}"
    password_check
    promised

    $ event list "lookup" && printf '%s\n' "${REPLY[@]}"
    lookup
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

#### event error

`event error` *message [exitlevel]* prints *message* to stderr and returns *exitlevel*, or 64 (`EX_USAGE`) if no *exitlevel* is given.  (You will still need to `return` or `exit` for this to have any further effect, unless `set -e` is in effect.)

````sh
    $ event error "This is an error" 127 >/dev/null
    This is an error
    [127]
````



### License

<p xmlns:dct="http://purl.org/dc/terms/" xmlns:vcard="http://www.w3.org/2001/vcard-rdf/3.0#">
  <a rel="license" href="http://creativecommons.org/publicdomain/zero/1.0/"><img src="https://licensebuttons.net/p/zero/1.0/80x15.png" style="border-style: none;" alt="CC0" /></a><br />
  To the extent possible under law, <a rel="dct:publisher" href="https://github.com/pjeby"><span property="dct:title">PJ Eby</span></a>
  has waived all copyright and related or neighboring rights to <span property="dct:title">bashup/events</span>.
This work is published from: <span property="vcard:Country" datatype="dct:ISO3166" content="US" about="https://github.com/bashup/events">United States</span>.
</p>