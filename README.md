# NPI
---
## Overview
NPI, or Nagios Passive Ingest, is a SaltStack engine that takes specific events off the minion event bus of the Nagios server and loads them into Nagios as passive check results. Specific events refer to events for which the engine has a parser to process the data of the event. All parsers live in the `npi_parsers` directory. To be made available to the engine, there needs to be an import statement and as well as an entry in the `parsers` dictionary in the `start` function of the `npi.py` file.

---
## Architecture
Generally, the way this is useful to me is through the use of SaltStack Beacons. I first write a beacon to check the status of something on a minion. The beacon sends Salt events to the master containing the status of that thing. The master receives the beacon event and through the use of the Salt Reactor, knows to relay this event to the Nagios minion over the event bus. The NPI engine reads the event once it arrives on the Nagios minion's event bus and checks to see if there is a parser available for that type of event.

---
## Example
As an example, lets take the parser found in [npi_parsers/salt_minion_heartbeat.py](https://github.com/alektant/npi). This parser is executed when events are received that have been generated by minions running the `salt_minion_heartbeat` beacon, that can be found in my [salt_beacons](https://github.com/alektant/salt_beacons) repo. One of those events will look similar to:

`{'salt_minion_heartbeat': 1472683942, 'id': 'endor.smartaleksolutions.com'}`

The parser will break the event apart and turn it into a Nagios passive check result which will look similar to:

`[1492122573] PROCESS_SERVICE_CHECK_RESULT;salt.smartaleksolutions.com;salt_minion_heartbeat;0;OK: last minion heartbeat: 1492122573|salt_minion_heartbeat=0;20;60`

Once the event has been transformed into a Nagios passive check result, it is written to the Nagios Command file. See the [Nagios website](https://assets.nagios.com/downloads/nagioscore/docs/nagioscore/3/en/passivechecks.html) for more on enabling passive checks and the command file.

### Nagios Server Minion Configuration
In order for SaltStack to recognize that it needs to start an engine on the Nagios minion, there needs to be a configuration file in `/etc/salt/minion.d/` that looks similar to:

```
engines:
  - npi:
      thresholds:
        salt_minion_heartbeat:
          critical: 60
          warning: 20
          ok: 0
      nagios_cmd_file: /var/spool/nagios/cmd/nagios.cmd
```

This tells SaltStack to run the `start` function of the `npi.py` engine file and pass it a list containing the data `thresholds` as well as `nagios_cmd_file`.

### Master Configuration
Since all events generated on minions are sent to the master, we need a SaltStack Reactor config that lets the master know which events should be forwarded on to our Nagios server. This configuration file must go in `/etc/salt/master.d/` and should look something like:

```
reactor:
  - 'salt/beacon/*/salt_minion_heartbeat/':
    - /srv/salt/_reactors/nagios_forward_event.sls
```

This tells the Salt master that events that come in with the tag `salt/beacon/*/salt_minion_heartbeat/` should be processed by the code in `/srv/salt/_reactors/nagios_forward_event.sls`

And so of course we need `/srv/salt/_reactors/nagios_forward_event.sls` and it should look like:
```
nagios_forward_event:
  local.event.fire:
    - tgt: nagios.smartaleksolutions.com
    - arg:
      - data: {{ data }}
      - tag: {{ tag }}
```