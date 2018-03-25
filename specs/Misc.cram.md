## Miscellaneous Cases and Error Handling

We want to test all this with `-eu`, to catch undefined variables and dangling fail statuses:

````sh
    $ source bashup.events; set -eu
````

We also want to make sure that validation errors work:

````sh
    $ ( event on foo.bar baz ) || echo [$?]
    Invalid event name 'foo.bar'
    [64]
````

Also, what happens if you don't pass anything to event on, .has, or .off?

````sh
    $ ( event on foo ) || echo [$?]
    foo: missing callback
    [64]

    $ ( event has ) || echo [$?]
    Invalid event name ''
    [64]

    $ ( event has x ) || echo [$?]
    [1]
    $ event on x echo y

    $ ( event has x ); echo [$?]
    [0]
````

Or don't give an event to fire or emit?

````sh
    $ ( event emit ) || echo [$?]
    Invalid event name ''
    [64]

    $ ( event fire ) || echo [$?]
    Invalid event name ''
    [64]
````

And what if you fire or emit an arg-limited event?

````sh
    $ event on foo/_ echo

    $ event emit foo/7 bar baz
    bar baz

    $ event fire foo/99 bar baz
    bar baz
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