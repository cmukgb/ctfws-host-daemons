#######################
The CtFwS MQTT Protocol
#######################

All numbers herein are base-10 encoded and devoid of leading zeros for ease
of parsing.  Unless otherwise indicated, numbers are non-negative.

MQTT messages should, unless otherwise indicated, be set persistent so that
devices that reboot or lose their connection will display the right thing upon
reconnection.  Most messages are designed to be idempotent, in the sense that
they carry timestamps of their veracity, so out-of-order delivery is partially
mitigated.  Timestamps are in UTC seconds which is easily obtained over the
network using (S)NTP.

Public, Centrally-set topics
############################

The public aspects of the game are set under ``ctfws/game`` (e.g.
``ctfws/game/config``).  We grant read-only views of these topics to guest
logins as well as our jail timers' users.

* ``config`` the string ``none`` or a whitespace-separated series of fields:

  * ``start_time`` -- UTC seconds indicating start state

  * ``setup_duration`` -- setup duration, in seconds

  * ``rounds`` -- number of rounds (intervals between jail breaks)

  * ``round_duration`` -- seconds per round

  * ``nflags`` -- number of flags per team

  * ``game_counter`` -- (integer) which game in a bunch is this?  Since often
    several are played in a night, it is likely useful to indicate to clients
    which game this is.  The value ``0`` may be interpreted as suppressing
    indication in the client; we ``1``-index games to be friendly to people.
    ;)

  * ``territory_config`` -- string; a site-specific descriptor of the
    territory configuration for this game.  We reserve ``-`` to mean
    that this field is being left unspecified despite later values (if any)
    being specified.

        .. note::

           The CMU devices expect the contents of this field to be either
           ``dw`` (for the red team defending doherty) or ``wd``.

  * any additional fields are to be ignored.

* ``endtime`` -- a single number, denoting UTC seconds of a
  forced game end.  If this is larger than the last ``starttime`` gotten
  in a ``config`` message, then the game is considered over.

* ``flags`` -- a whitespace-separated text field.  The first field is a
  UTC-seconds timestamp; subsequent fields are either the string ``?`` or:

  * ``red`` -- red team flag capture count (int, negatives OK)
 
  * ``yel`` -- yellow team flag capture count (int, negatives OK)

  * any additional fields are to be ignored.

* ``message`` -- Message to be displayed everywhere.  This and
  all other ``message/#`` topics have a UTC-seconds timestamp followed by
  whitespace before the message body.  These permit messages from previous
  games to be suppressed, should they end up resident on the MQTT broker.

* ``message/player`` -- Message to be displayed specifically
  to players' interfaces (i.e., web, Android).

* ``message/jail`` -- Message to be displayed specifically at
  jail glyph units.

* ``message/jail/#`` -- Reserved for messages directed to a particular jail
  glyph; at present, our devices do not subscribe to these endpoints.

* ``messagereset`` -- A single number, denoting UTC seconds
  before which messages should not be displayed.  This is useful in the
  event that the judges send out an incorrect message.

There are some additional public topics not under ``ctfws/game`` as they do not
pertain to a particular game, but rather to the world more generally:

* ``ctfws/timesync`` -- a single number, denoting UTC seconds at the time of
  its publication.  The head judge's computer or the broker should publish to
  this topic periodically (every minute?) to assist clients in measuring their
  clock skew.  Clients must ignore retained messages on this topic, as they are
  by definition stale; messages should be published with QoS 0: a delayed
  message is worse than no message.  (See ``daemons/timesync-during-game`` for
  our manager for this topic.)

Rule Documentation URLs
=======================

Under ``ctfws/rules``, some URLs to user-visible documentation are published.
Messages for here are composed of whitespace-separated fields:

  * ``url`` -- a URL whence the document may be downloaded;
               (spaces are to be URL-encoded, naturally enough).
  * ``time`` -- (integer) UTC seconds at which the handbook was last modified
  * ``sha256`` -- A hex encoding of the SHA256 of the handbook file.

The ``time`` and ``sha256`` fields are to assist clients in suppressing fetches
when initially subscribing or when the publication daemons spuriously announce
the old result (which they might, for example, on startup).

The following topics are defined:

  * ``ctfws/rules/handbook/html`` -- a single-HTML-page version of the handbook.
  * ``ctfws/rules/cheatsheet/pdf`` -- a PDF version of the "cheatsheet",
    a half-page or so summary of useful material.

See ``daemons/inotifywait-handbook`` for a primitive, but sufficient,
implementation of a script to maintain these topics' values.

Private, Centrally-set topics
#############################

Some information is used internally by the judges; it is not intended for
player view.  These are set under the prefix ``ctfws/judge``.

* ``flags`` a whitespace-separated text string with two integer
  fields:

  * ``timestamp`` -- the UNIX time of this message

  * ``red`` -- An integer, encoding the set of red flags that have been
    captured.  The 1 bit corresponds to flag A, 2 to flag B, etc.  That is,
    if yellow has captured red flags C and H, this field has the value 132.

  * ``yellow`` -- As above, but for the set of yellow flags that have been
    captured.

  .. note:: 

     This is just a proposal at the moment.  There's also the suggestion
     that ``flags`` be a whitespace-separated text string with two text fields,
     consisting of strings of letters for which flags have been captured,
     e.g., "CDH AB" if red has captured those three yellow flags and yellow
     those two red ones.  Lord help
     us if we ever have more than 94 (the printable ASCII set, minus space)
     flags in play.  The field need not be sorted.

Some information is communicated from the judges to devices directly.  (In the
following, ``$DEVICENAME`` refers to the MQTT user identity given to the device
in question.)  While there is no harm in players seeing this information, it is
unlikely to be of interest:

* ``ctfws/devc/$DEVICENAME/location``  Reserved for device-specific
  configuration, in particular for parsing ``ctfws/game/config``'s
  ``territory_config`` for display.

  .. note::

     The CMU devices are likely to use either the character `d` or `w` to
     indicate the location of the device.

* ``ctfws/devc/$DEVICENAME/role``  Reserved for device-specific configuration.
  Jail timers should either not have this set or should use the reserved value
  ``jail``; other devices may be assigned other roles, should we ever branch out.

Device-set topics
#################

Devices get to send messages to some topics, too, to provide centralized
view of the world.

* ``ctfws/dev/$DEVICENAME/beat``

  * one of ``alive``, ``beat``, or ``dead``
  * ``time`` (UNIX time, from local clock)
  * ``ap`` (MAC addr)
  * ``clocksource`` (structured; see below)
  * ``timeskew`` (UNIX seconds)
  * any additional fields are to be ignored.

  The device should use this as its last will and testament (LWT) topic, with
  ``dead`` as the message, published QoS 2 and retained.  When a client
  successfully connects to the broker, it should publish ``alive``.
  Thereafter, it should publish ``beat`` messages every minute.

  All fields other than the first are optional, with ``-`` being reserved for
  the case of optional fields being elided but later fields being specified.

  Note that ``dead`` messages will not have a timestamp (or AP MAC address) due
  to the mechanics of MQTT: the LWT message must be known at client connection
  time and cannot be updated during the client's operation.  Historically,
  ``alive`` messages have not included a timestamp either, perhaps to allow
  SNTP synchronization during the first beat period.

  In order to observe the behavior of clocks in the field, we have introduced
  two new values, ``clocksource`` and ``timeskew``.  Both are optional, with
  ``-`` defined, as usual, to indicate that the value is being skipped to
  transmit later fields.  The ``clocksource`` field conveys which source of
  time was used to last set the local clock, possibly the time value set, and
  possibly the relative difference from source and local clock; the value is
  reported as a source string (``sntp`` or ``mqtt`` are likely candidates),
  an optional ``@`` followed by the UNIX seconds reported by the source, and
  an optional ``+`` followed by the (signed) UNIX seconds delta from local
  clock.  If present, the ``timeskew`` captures the number of
  seconds difference between the local clock and the last ``timesync`` message
  at time of receipt of the latter; if the ``timesync`` message is itself the
  clock source, then this will equal the delta reported in ``clocksource``.

ACL
###

The file ``broker/acl`` gives a suitable mosquitto-compatible broker ACL for
the topic tree given above.
