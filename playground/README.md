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

Note: 
* while it didn't specifically help me in this case, this old [conference paper on using SWIG with python](http://www.gti.bh/Library/assets/171-pytutorial98.pdf) was very interesting.

#### DEAD END

Let's go back and try commenting out references to `ezsp` and see if we can get this running without zigbee data first.

Ugh, it's too integrated into the code. It's everywhere. In constructors. etc.

#### DEAD END

Maybe I should go back to trying to get ezsp to import correctly. If I could do that, I might be able to get this working with the $50 zigbee attachment from the same vendor as the one in the bridge.

______

2020-07-06, 11pm. back at it.

Aha! I didn't have `_ezsp.so` in the `playground` directory. Moving it there changed the error from

`ImportError: No module named _ezsp`

to

`ImportError: ./_ezsp.so: cannot open shared object file: No such file or directory`

which is weird, b/c the file is now there. Maybe the permissions are wrong? (nope, they're 755.)

Per [this stackoverflow Q](https://stackoverflow.com/questions/1099981/why-cant-python-find-shared-objects-that-are-in-directories-in-sys-path), I tried setting LD_LIBRARY_PATH and/or updating `/etc/ld.so.conf` and then running `ldconfig`, but that didn't help. Changed them back.

Man, every answser on the internet to this question says it's something about the LD_LIBRARY_PATH and the linker not knowing where to look, so I must be close. :/

TODO: Read more about [linux shared libraries](http://tldp.org/HOWTO/Program-Library-HOWTO/shared-libraries.html)

______

(note to self: I probably need to have python and/or the system looking at the proper lib directory, since ezsp probably won't be the only linking error)

_____
2020-07-11

So, it's not a linking error, the problem is that `_ezsp.so` isn't compiled in a way that the pi's ARM processor can read it. (It was compiled for _an_ ARM processor, though! why doesn't it work here?)

### `ldd`

`ldd` output, on `_ezsp.so` and a random so file I grabbed from the pi's python library

````
$ ldd _ezsp.so
	not a dynamic executable

$ ldd /usr/lib/python2.7/lib-dynload/dbm.arm-linux-gnueabihf.so
	linux-vdso.so.1 (0xbec81000)
	/usr/lib/arm-linux-gnueabihf/libarmmem-${PLATFORM}.so => /usr/lib/arm-linux-gnueabihf/libarmmem-v7l.so (0xb6ece000)
	libdb-5.3.so => /usr/lib/arm-linux-gnueabihf/libdb-5.3.so (0xb6d34000)
	libpthread.so.0 => /lib/arm-linux-gnueabihf/libpthread.so.0 (0xb6d0a000)
	libc.so.6 => /lib/arm-linux-gnueabihf/libc.so.6 (0xb6bbc000)
	/lib/ld-linux-armhf.so.3 (0xb6ef6000)
  ````

  The rest of the binaries in the dump likely have the same problem. For example, I tried to run the python executable in the dump and got "no such file or directory" (despite the file very much existing), the same error we saw when trying to import `_ezsp.so`.

  ````
  $ /home/pi/dev/berg-bridge-dump/usr/bin/python2.7 --version
  /home/pi/dev/berg-bridge-dump/usr/bin/python2.7: No such file or directory
  ````

### `readelf`
  Some googling led to using `readelf` to figure out more about what architecture the binaries were compiled for.

  Here's the output for the pi's python executable (first), and the python executable included in the dump. They look almost the same. I'm not sure which parts of the output I should be comparing. :/

#### rasbian's python2.7 executable

````
readelf -h /usr/bin/python2.7
ELF Header:
  Magic:   7f 45 4c 46 01 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF32
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              EXEC (Executable file)
  Machine:                           ARM
  Version:                           0x1
  Entry point address:               0x4ace4
  Start of program headers:          52 (bytes into file)
  Start of section headers:          2983696 (bytes into file)
  Flags:                             0x5000400, Version5 EABI, hard-float ABI
  Size of this header:               52 (bytes)
  Size of program headers:           32 (bytes)
  Number of program headers:         9
  Size of section headers:           40 (bytes)
  Number of section headers:         28
  Section header string table index: 27
````

#### the python2.7 executable from the dump file
````
$ readelf -h /home/pi/dev/berg-bridge-dump/usr/bin/python2.7
ELF Header:
  Magic:   7f 45 4c 46 01 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF32
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              EXEC (Executable file)
  Machine:                           ARM
  Version:                           0x1
  Entry point address:               0x8474
  Start of program headers:          52 (bytes into file)
  Start of section headers:          1896 (bytes into file)
  Flags:                             0x5000002, Version5 EABI, <unknown>
  Size of this header:               52 (bytes)
  Size of program headers:           32 (bytes)
  Number of program headers:         6
  Size of section headers:           40 (bytes)
  Number of section headers:         20
  Section header string table index: 19
  ````

## Next steps
* try to get a copy of ezsp from si labs to see if I can compile it for the pi?
* try decompiling/cross-compiling `_ezsp.so` (ugh don't wanna)

Installed SiLabs' IDE, "Simplicity Studio", but it wouldn't run on my mac. Filed the following question in their forums: [Can't run on macOS Catalina](https://www.silabs.com/community/software/simplicity-studio/forum.topic.html/can_t_run_on_macoscatalina-vTZa).
____


