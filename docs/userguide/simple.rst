==================
 Simple Interface
==================

.. contents::
    :local:


:mod:`kombu.simple` is a simple interface to AMQP queueing.
It is only slightly different from the :class:`~Queue.Queue` class in the
Python Standard Library, which makes it excellent for users with basic
messaging needs.

Instead of defining exchanges and queues, the simple classes only requires
two arguments, a connection channel and a name. The name is used as the
queue, exchange and routing key. If the need arises, you can specify
a :class:`~kombu.entity.Queue` as the name argument instead.

In addition, the :class:`~kombu.connection.BrokerConnection` comes with
shortcuts to create simple queues using the current connection::

    >>> queue = connection.SimpleQueue("myqueue")
    >>> # ... do something with queue
    >>> queue.close()


This is equivalent to::

    >>> from kombu import SimpleQueue, SimpleBuffer

    >>> channel = connection.channel()
    >>> queue = SimpleBuffer(channel)
    >>> # ... do something with queue
    >>> channel.close()
    >>> queue.close()

Connections and transports
==========================

To send and receive messages you need a transport and a connection.
There are several transports to choose from (amqplib, pika, redis, in-memory),
and you can even create your own. The default transport is amqplib.

Create a connection using the default transport::

    >>> from kombu import BrokerConnection
    >>> connection = BrokerConnection()

The connection will not be established yet, as the connection is established
when needed. If you want to explicitly establish the connection
you have to call the :meth:`~kombu.connection.BrokerConnection.connect`
method::

    >>> connection.connect()

This connection will use the default connection settings, which is using
the localhost host, default port, username ``guest``,
passowrd ``guest`` and virtual host "/". A connection without arguments
is the same as::

    >>> BrokerConnection(hostname="localhost",
    ...                  userid="guest",
    ...                  password="guest",
    ...                  virtual_host="/",
    ...                  port=6379)

The default port is transport specific, for AMQP transports this is 6379.

Other fields may also have different meanings depending on the transport
used. For example, the ``virtual_host`` argument is used as the database
number in the Redis transport.

See the reference documentation for
:class:`~kombu.connection.BrokerConnection` for a full list of arguments
supported.


Sending and receiving messages
==============================

The simple interface defines two classes; :class:`~kombu.simple.SimpleQueue`,
and :class:`~kombu.simple.SimpleBuffer`. The former is used for persistent
messages, and the latter is used for transient, buffer-like queues.
They both have the same interface, so you can use them interchangeably.

Here is an example using the :class:`~kombu.simple.SimpleQueue` class
to produce and consume logging messages:

.. code-block:: python

    from socket import gethostname
    from time import time

    from kombu import BrokerConnection


    class Logger(object):

        def __init__(self, connection, queue_name="log_queue",
                serializer="json", compression=None):
            self.queue = connection.SimpleQueue(self.queue_name)
            self.serializer = serializer
            self.compression = compression

        def log(self, message, level="INFO", context={}):
            self.queue.put({"message": message,
                            "level": level,
                            "context": context,
                            "hostname": socket.gethostname(),
                            "timestamp": time()},
                            serializer=self.serializer,
                            compression=self.compression)

        def process(self, callback, n=1, timeout=1):
            for i in xrange(n):
                log_message = self.queue.get(block=True, timeout=1)
                entry = log_message.payload # deserialized data.
                callback(entry)
                log_message.ack() # remove message from queue

        def close(self):
            self.queue.close()


    if __name__ == "__main__":
        connection = BrokerConnection(hostname="localhost",
                                      userid="guest",
                                      password="guest",
                                      virtual_host="/")
        logger = Logger(connection)

        # Send message
        logger.log("Error happened while encoding video",
                   level="ERROR",
                   context={"filename": "cutekitten.mpg"})

        # Consume and process message

        # This is the callback called when a log message is
        # received.
        def dump_entry(entry):
            date = datetime.fromtimestamp(entry["timestamp"])
            print("[%s %s %s] %s %r" % (date,
                                        entry["hostname"],
                                        entry["level"],
                                        entry["message"],
                                        entry["context"]))

        # Process a single message using the callback above.
        logger.process(dump_entry, n=1)

        logger.close()