
Creating Messages
-----------------

To create a FIX message, first create an instance of the FixMessage class.

.. index:: FixMessage
.. code-block:: python
    :linenos:

    message = simplefix.FixMessage()

.. index:: header

You can then add fields to the message as required.  You should add the
standard header tags 8, 34, 35, 49, 52, and 56 to all messages, unless
you're deliberately creating a malformed message for testing or similar.

Simple Fields
.............

.. index:: append_pair

For most tags, using ``append_pair()`` is the easiest way to add a field
to the message.

.. code-block:: python
    :linenos:

    message.append_pair(1, "MC435967")
    message.append_pair(54, 1)
    message.append_pair(44, 37.0582)

Note that any type of value can be used: it will be explicitly converted
to a string before encoding the message.

.. index:: BeginString, BodyLength, MsgType, Checksum

With a few exceptions, the message retains the order in which fields are
added.  The exceptions are that fields BeginString (8), BodyLength (9),
MsgType (35), and Checksum (10) are encoded in their required locations,
regardless of what order they were added to the Message.

Header Fields
.............

The ``FixMessage`` class does not distinguish header fields from body
fields, with one exception.

.. index:: append_pair, header

To enable fields to be added to the FIX header after body fields have
already been added, there's an optional keyword parameter to the
``append_pair()`` method (and other append field methods).  If this
``header`` parameter is set to ``True``, the field is inserted after
any previously added header fields, starting at the beginning of the
message.

.. index:: MsgSeqNum, SendingTime

This is normally used for setting things like MsgSeqNum (34) and
SendingTime (52) immediately prior to encoding and sending the message.

.. code-block:: python
    :linenos:

    message.append_pair(8, "FIX.4.4")
    message.append_pair(35, 0)
    message.append_pair(49, "SENDER")
    message.append_pair(56, "TARGET")
    message.append_pair(112, "TR0003692")
    message.append_pair(34, 4684, header=True)
    message.append_time(52, header=True)

In the example above, field 34 would be inserted at the beginning of
the message.  After encoding, the order of fields would be: 8, 9, 35,
34, 52, 49, 56, 112, 10.

It's not necessary, but field 49 and 56 could also be written with
``header`` set ``True``, in which case, they'd precede 34 and 52 when
encoded.

.. index:: append_utc_timestamp

See ``append_utc_timestamp()`` below for details of that method.

Pre-composed Pairs
..................

.. index:: append_string, append_strings

In some cases, your FIX application might have the message content
as pre-composed "tag=value" strings.  In this case, as an optimisation,
the ``append_string()`` or ``append_strings()`` methods can be used.

.. code-block:: python
    :linenos:

    BEGIN_STRING = "8=FIX.4.2"
    STR_SEQ = ["49=SENDER", "56=TARGET"]

    message.append_string(BEGIN_STRING, header=True)
    message.append_strings(STR_SEQ, header=True)

As with ``append_pair()``, note that these methods have an optional
keyword parameter to ensure that their fields are inserted before
body fields.

Timestamps
..........

.. index:: UTCTimestamp, UTCTimeOnly, TZTimestamp, TZTimeOnly

The FIX protocol defines four time types: UTCTimestamp, UTCTimeOnly,
TZTimestamp, and TZTimeOnly.  Field values of these types can be added
using dedicated functions, avoiding the need to translate and format
time values in the application code.

.. index:: append_utc_timestamp, append_tz_timestamp
.. index:: append_utc_time_only, append_tz_time_only
.. code-block:: python
    :linenos:

    message.append_utc_timestamp(52, precision=6, header=True)
    message.append_tz_timestamp(1132, my_datetime)
    message.append_utc_time_only(1495, start_time)
    message.append_tz_time_only(1079, maturity_time)

.. index:: time, datetime

The first parameter to these functions is the field's tag number.  The
second parameter is optional: if None or not supplied, it defaults to the
current time, otherwise it must be a Unix epoch time (like from
``time.time()``), or a ``datetime`` instance.

.. index:: precision, second, milliseconds, microseconds, header

There are two keyword parameters: ``precision`` which can be 0 for just
seconds, 3 for milliseconds, or 6 for microseconds; and ``header`` to
insert this field in the header rather than the body.

In addition, there are a set of methods for creating correctly formatted
time only values from their components:

.. index:: append_utc_time_only_parts, append_tz_time_only_parts
.. code-block:: python
    :linenos:

    message.append_utc_time_only_parts(1495, 7, 0, 0, 0, 0)
    message.append_tz_time_only_parts(1079, 20, 0, 0, offset=-300)

As usual, the first parameter to these functions is the field's tag number.
The next three parameters are the hour, minute, and seconds of the time value,
followed by optional milliseconds and microseconds values.

.. index:: timezone

The timezone for the TZTimeOnly field is set using an offset value, the
number of minutes east of UTC.  Thus CET will be offset 60 minutes, and
New York offset -240 minutes (four hours west).

Finally, remember that time fields can always be set using a string value
if the application already has the value in the correct format or prefers
to manage the formatting itself.

Repeating Groups
................

.. index:: repeating group

There is no specific support for creating repeating groups in FixMessages.
The count field must be appended first, followed by the group's member's
fields.

Consequently, it's not an error to append two fields with the same tag,
but note that the count fields are not added automatically.

Data Fields
...........

.. index:: data, raw data

There are numerous defined fields in the FIX protocol that use the *data*
type.  These fields consist of two parts: a length, which must come first,
immediately followed by the value field, whose value may include the ASCII
SOH character, the ASCII NUL character, and in fact any 8-bit byte value.

.. index:: append_data

To append a data field to a message, the ``append_data()`` method can be
used.  It will correctly add both the length field and the value field.

.. code-block:: python
    :linenos:

    message.append_data(95, 96, "RAW DATA \x00\x01 VALUE")

which will result in the FIX message content (where | represents the SOH):

.. epigraph::

    95=17|96=RAW DATA \\x00\\x01 VALUE|
