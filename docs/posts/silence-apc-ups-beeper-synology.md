---
authors: [stimmerman]
comments: true
categories:
  - Synology
date: 2024-08-12
tags:
  - apc
  - ups
  - synology
  - dsm
---
# Howto silence APC UPS alarm sound that's attached to Synology
I'm using an APC Back-UPS ES 700G at home to keep my NAS and NUC running and do a safe shutdown in case of power failure. It's attached with an USB cable to my Synology NAS running DSM 7.x so the NAS can check the status of the UPS and act on it.

Unfortunately the battery of the UPS failed, causing it to make a loud alarm sound which can't be silenced on the UPS itself with a button. Here's how it can be silenced from the NAS.

<!-- more -->

!!! note "Note"
    This post based on the post from [Moshi Bin](https://moshib.in/posts/disable-ups-beeper-synology/). I've adapted it to work with DSM 7.x and with **the only goal to just silcence the beeper**, the quickest way possible until the replacement battery comes.

## NUTS (Network UPS Tools)

DSM runs [NUTS (Network UPS Tools)](https://networkupstools.org/) as tool to manage an attached UPS. The NUTS software is set up as client-server model and normally comes with a tool called `upscmd` which can be used to change settings for the UPS. On DSM this tool unfortunately has been left out by Synology. But we can still pretend to be a client to NUTS by using telnet and change the setting.

### Update permissions

Login to the NAS with your admin account and update the file containing the users and permissions in `/etc/ups/upsd.users`
```
admin@NAS:~$ sudo vi /etc/ups/upsd.users
```

At the bottom you should see this config for the default `monuser`:
```
# MONITOR myups@localhost 1 upsmon pass master  (or slave)
[monuser]
        password = secret
        upsmon master
```

Add the `actions` and `instcmds` lines as shown below:

```
# MONITOR myups@localhost 1 upsmon pass master  (or slave)
[monuser]
        password = secret
        upsmon master
        actions = SET
        instcmds = beeper.disable
```

Save the file, and reload the `upsd` service:
```
admin@NAS:~$ sudo upsd -c reload
Network UPS Tools upsd DSM7-2-1-NewModel-repack-64570-230831
admin@NAS:~$
```

### Telnet using python

There's no telnet client available on DSM, we can use Python and the builtin `telnetlib` to do the same. Create a new file like `disable_ups_beep.py`
```
admin@NAS:~$ vi disable_ups_beep.py
```
And paste the this code
```
#!/usr/bin/env python
from telnetlib import Telnet

with Telnet("localhost", 3493) as tn:
    tn.write(b"USERNAME monuser\n")
    tn.read_until(b"OK")
    tn.write(b"PASSWORD secret\n")
    tn.read_until(b"OK")
    tn.write(b"INSTCMD ups beeper.disable\n")
    tn.read_until(b"OK")
    tn.write(b"LOGOUT\n")
    print(tn.read_all())
```
Execute the script and you should see a similar output like this (and finally the sound will stop):
```
admin@NAS:~$ ./disable_ups_beep.py
b'\nOK Goodbye\n'
admin@NAS:~$
```
