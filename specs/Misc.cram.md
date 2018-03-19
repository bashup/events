## Miscellaneous Cases and Error Handling

We want to test all this with `-u`, to catch undefined variables references:

    $ source bashup.events; set -u

We also want to make sure that validation errors work:

    $ ( event.on foo.bar baz )
    Invalid event name 'foo.bar'
    [64]

Also, what happens if you don't pass anything to event.on, .has, or .off?

    $ ( event.on foo )
    foo: missing callback
    [64]

    $ ( event.has )
    Invalid event name ''
    [64]

Or don't give an event to fire or emit?

    $ ( event.emit )
    Invalid event name ''
    [64]

    $ ( event.fire )
    Invalid event name ''
    [64]

And what if you fire or emit an arg-limited event?

    $ event.on foo echo

    $ event.emit foo@7 bar baz
    bar baz

    $ event.fire foo@99 bar baz
    bar baz
