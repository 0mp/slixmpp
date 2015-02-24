.. _using_asyncio:

=============
Using asyncio
=============

Block on IQ sending
~~~~~~~~~~~~~~~~~~~

:meth:`.Iq.send` now accepts a ``coroutine`` parameter which, if ``True``,
will return a coroutine waiting for the IQ reply to be received.

.. code-block:: python

    result = yield from iq.send(coroutine=True)

XEP plugin integration
~~~~~~~~~~~~~~~~~~~~~~

Many XEP plugins have been modified to accept this ``coroutine`` parameter as
well, so you can do things like:

.. code-block:: python

    iq_info = yield from self.xmpp['xep_0030'].get_info(jid, coroutine=True)


Running the event loop
~~~~~~~~~~~~~~~~~~~~~~

:meth:`.XMLStream.process` is only a thin wrapper on top of
``loop.run_forever()`` (if ``timeout`` is provided then it will
only run for this amount of time).

Therefore you can handle the event loop in any way you like
instead of using ``process()``.


Examples
~~~~~~~~

Blocking until the session is established
-----------------------------------------

This code blocks until the XMPP session is fully established, which
can be useful to make sure external events aren’t triggering XMPP
callbacks while everything is not ready.

.. code-block:: python

    import asyncio, slixmpp

    client = slixmpp.ClientXMPP('jid@example', 'password')
    client.connected_event = asyncio.Event()
    callback = lambda event: client.connected_event.set()
    client.add_event_handler('session_start', callback)
    client.connect()
    loop.run_until_complete(event.wait())
    # do some other stuff before running the event loop, e.g.
    # loop.run_until_complete(httpserver.init())
    client.process()


Use with other asyncio-based libraries
--------------------------------------

This code interfaces with aiohttp to retrieve two pages asynchronously
when the session is established, and then send the HTML content inside
a simple <message>.

.. code-block:: python

    import asyncio, aiohttp, slixmpp

    @asyncio.coroutine
    def get_pythonorg(event):
        req = yield from aiohttp.request('get', 'http://www.python.org')
        text = yield from req.text
        client.send_message(mto='jid2@example', mbody=text)

    @asyncio.coroutine
    def get_asyncioorg(event):
        req = yield from aiohttp.request('get', 'http://www.asyncio.org')
        text = yield from req.text
        client.send_message(mto='jid3@example', mbody=text)

    client = slixmpp.ClientXMPP('jid@example', 'password')
    client.add_event_handler('session_start', get_pythonorg)
    client.add_event_handler('session_start', get_asyncioorg)
    client.connect()
    client.process()


Blocking Iq
-----------

This client checks (via XEP-0092) the software used by every entity it
receives a message from. After this, it sends a message to a specific
JID indicating its findings.

.. code-block:: python

    import asyncio, slixmpp

    class ExampleClient(slixmpp.ClientXMPP):
        def __init__(self, *args, **kwargs):
            slixmpp.ClientXMPP.__init__(self, *args, **kwargs)
            self.register_plugin('xep_0092')
            self.add_event_handler('message', self.on_message)

        @asyncio.coroutine
        def on_message(self, event):
            # You should probably handle IqError and IqTimeout exceptions here
            # but this is an example.
            version = yield from self['xep_0092'].get_version(message['from'],
                                                              coroutine=True)
            text = "%s sent me a message, he runs %s" % (message['from'],
                                                         version['software_version']['name'])
            self.send_message(mto='master@example.tld', mbody=text)

    client = ExampleClient('jid@example', 'password')
    client.connect()
    client.process()


