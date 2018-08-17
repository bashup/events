## Miscellaneous Cases and Error Handling

We want to test all this with `-eu`, to catch undefined variables and dangling fail statuses:

````sh
    $ source bashup.events; set -eu
````

e.g. What happens if you don't pass a callback to event on, .has, or .off?

````sh
    $ ( event on foo ) || echo [$?]
    foo: missing callback
    [64]

    $ ( event on ) || echo [$?]
    : missing callback
    [64]

    $ ( event has x ) || echo [$?]
    [1]

    $ event on x echo y

    $ ( event has x ); echo [$?]
    [0]

````

Or don't give an event to fire or emit?  (they're treated as an empty string)

````sh
    $ ( event has ) || echo [$?]
    [1]

    $ event on "" echo empty-string event

    $ ( event has ) && echo yes
    yes

    $ ( event emit ) || echo [$?]
    empty-string event

    $ ( event fire ) || echo [$?]
    empty-string event

````

And what if you try to pass an arg count to something other than has/on/off? (they're treated as arguments)

````sh
    $ event on foo @_ echo

    $ event emit foo @7 bar baz
    @7 bar baz

    $ event fire foo @_ bar baz
    @_ bar baz
````

Or an invalid arg indicator to has/on/off? (they're considered part of the command)

````sh
    $ event on bar @9.2 quiz
    $ event emit bar || echo [$?]
    */bashup.events: line ?: @9.2: command not found (glob)
    [127]
````

Or try to do something with an already-resolved promise:

````sh
# Duplicate emit/fire/resolve/any/all produces code 70 (EX_SOFTWARE):

    $ event resolve "promised" other
    $ set +e

    $ event resolve "promised" other
    event "promised" already resolved
    [70]

    $ event fire "promised" other
    event "promised" already resolved
    [70]

    $ event emit "promised" other
    event "promised" already resolved
    [70]

    $ event any "promised" other
    event "promised" already resolved
    [70]

    $ event all "promised" other
    event "promised" already resolved
    [70]
````

Also, "event once" shouldn't leave anything behind in the event, and should handle  various argument counts (the `@_` case is tested in the README):

````sh
    $ event once foobar echo baz
    $ event emit foobar
    baz

    $ event has foobar || echo nope
    nope

    $ event once foobar @2 echo fish
    $ event emit foobar baz spam thingy
    fish baz spam

    $ event has foobar || echo nope
    nope
````