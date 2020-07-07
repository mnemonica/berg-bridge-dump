Steps taken so far:
- created playground dir
- renamed files from `*.pyc.py` to `*.py`
- commented out decompilation status line in each file (starting with `+++ okay decompyling`)
- made all files executable

for the commands below,
- current `python`/`pip` version is `2.7.16`
- current `python3`/`pip3` version is `3.7.3`

running on raspberry pi 4, 4GB, "Raspbian GNU/Linux 10 (buster)"

### Run 1: `python engtest.py`
```
Traceback (most recent call last):
  File "engtest.py", line 3, in <module>
    import linux_hub
  File "/home/pi/dev/berg-bridge-dump/playground/linux_hub.py", line 9, in <module>
    import pynetlinux
ImportError: No module named pynetlinux
```
so let's install pynetlinux

`pip install pynetlinux`

### Run 2
```
Traceback (most recent call last):
  File "engtest.py", line 4, in <module>
    import api
  File "/home/pi/dev/berg-bridge-dump/playground/api.py", line 10, in <module>
    import ezsp
  File "/home/pi/dev/berg-bridge-dump/playground/ezsp.py", line 22, in <module>
    _ezsp = swig_import_helper()
  File "/home/pi/dev/berg-bridge-dump/playground/ezsp.py", line 12, in swig_import_helper
    import _ezsp
ImportError: No module named _ezsp
```

google says EZSP refers to the "Zigbee EZSP (EmberZNet Serial Protocol) interface". `ezsp.py` in this directory is trying to import `_ezsp.so` compiled library in this directory. 

I'm going to noodle on this for a while, but it could instead be a good idea to comment out any reference to `ezsp.py` to see how we could get this running _without_ the zigbee wireless to start.

Update: 
- copied `ezsp.pyc` and `._ezsp.pyc` from `usr/local/bergcloud-bridge`; it did not help
- tried adding env var with `export LD_LIBRARYPATH=.` based on [this q on the pi forums](https://www.raspberrypi.org/forums/viewtopic.php?t=190581). It didn't help.

#### DEAD END

Let's go back and try commenting out references to `ezsp` and see if we can get this running without zigbee data first.

Ugh, it's too integrated into the code. It's everywhere. In constructors. etc.

#### DEAD END

Maybe I should go back to tryign to get ezsp to import correctly. If I could do that, I might be able to get this working with the $50 zigbee attachment from the same vendor as the one in the bridge.

