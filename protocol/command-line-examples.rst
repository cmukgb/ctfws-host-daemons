These are kept for developer reference; the web interface offered by
https://github.com/cmukgb/ctfws-timer-web (and in particular
http://timer.cmukgb.org/judge/) is a vastly superior experience.

For the sake of simplicity in the below examples, set::

  M=(-h $MQTT_SERVER -u ctfwsmaster -P $CTFWSMASTER_PASSWD -q 2)

To watch what's going on in the world::

  mosquitto_sub "$M[@]" -t ctfws/\# -v

To send MQTT messages, try variants of these.  Note that in all cases, we
set messages persistent so that devices that (re)connect mid-way into a game
get the latest messages automatically.

* To start the 2nd game now, with 16 minutes of setup, 4 x 15 minute rounds, 10 flags, with
  red team defending Wean hall and the yellow team defending Doherty hall::

    mosquitto_pub "$M[@]" -t ctfws/game/flags -r -m `date +%s`' 0 0'
    mosquitto_pub "$M[@]" -t ctfws/game/config -r -m `date +%s`' 960 4 900 10 2 wd'

* To post information (The messages must have date stamps on the front!)::

    mosquitto_pub "$M[@]" -t ctfws/game/flags -r -m `date +%s`' 1 2'
    mosquitto_pub "$M[@]" -t ctfws/game/message -r -m `date +%s`' Red team captured a flag!'

* Note that you can deliberately hide the flag scores, if you like, by
  publishing ``?`` to the ``/flags`` topic::

    mosquitto_pub "$M[@]" -t ctfws/game/flags -r -m `date +%s`' ?'

* To end a game::

    mosquitto_pub "$M[@]" -t ctfws/game/endtime -r -m `date +%s`
