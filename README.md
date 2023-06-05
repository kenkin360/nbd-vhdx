# nbx-vhdx
A systemd service connects vhd/vhdx as NBDs and BitLocker partitions on startup

* Prerequisites: 

  This tool utilizes `qemu-nbd` for connecting disk image as NBDs and `cryptsetup` for operations on BitLocker partitions, they are required. `blkid` plays an important role in the script, it is also required. The rest commands, for example `mount/unount` are likely pre-installed in all distros. 

* install/uninstall: 

  Run `install-sh` or `uninstall-sh` along with `nbd-vhdx` and `nbd-vhdx.service` to install/uninstall the service and shell script executable. 
  
* Run in command line: 

  Run `nbd-vhdx` with no argument to see the valid commands. 
  
* /etc/vhdxtab

  The `/etc/vhdxtab` file describes disk images and BitLocker partitions that are set up during system boot. The format was taken from `/etc/crypttab` and an extra field for the filename of disk image was introduced. There are things need to be mentioned: 

  1. The first field (namely `target name`) is ignored as the BitLocker partitions will be mapped like `/dev/mapper/bitlk-<uuid>` where `<uuid>` would be the actual partition UUID. No user define name required (and not supported). 

  2. The forth field (namely `options`) is ignored as it takes only `bitlk` type and no extra option is supported. 

  3. If you just want to connect a disk image as an NBD without mapping any BitLocker partition, leave the first four fields with `-`

  4. If there are more BitLocker partitions in the same disk image, add each entry with the same filename. That said, the same filename can appear in multiple entries, and will be connected as only one NBD. 
