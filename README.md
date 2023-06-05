# nbx-vhdx
A systemd service connects vhd/vhdx as NBDs and BitLocker partitions on startup

* Prerequisites: 

  This tool utilizes `qemu-nbd` for connecting disk image as NBDs and `cryptsetup`(version 2.6.1) for operations on BitLocker partitions, they are required. `blkid` plays an important role in the script, it is also required. The rest commands, for example `mount/unount` are likely pre-installed in all distros. 

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

  Here's an example: 
  ```
  # <target name>	<source device>		<key file>	<options>
  - PARTUUID=d3eed7e3-01 /root/d3eed7e3-01.fvek bitlk /media/bin/bitlocker-test.vhdx
  - PARTUUID=bace267f-01 /root/bace267f-01.fvek bitlk /media/bin/vhdx-test.vhdx
  - - - - /media/bin/bitlk-test.vhd
  - PARTUUID=bace284a-01 /root/bace284a-01.fvek bitlk /media/bin/bitlk-test.vhd
  ```
    The third and forth entry suggested the same file are actually redundant but what if we sometimes want to passthrough the entire device to a vm that `bitlk-test.vhd` is connected to without mapping any BitLocker partition in it? We may just comment the forth entry out. On the other hand if there are more non-BitLocker partitions in `bitlk-test.vhd` we want to mount and describe with `/etc/fstab`, we may choise from entry thee - not to map the BitLocker partition -or- entry four - mount them altogether, then we may leave them as is or comment out them on our demand. 

* /etc/fstab

    We would just describe partitions to mount on startup in `/etc/fstab` as usual. The following is for an example: 
    
    ```
    UUID=xxxxxxxxxxxxxxxx /media/bin ntfs-3g defaults,nodev,nosuid,locale=zh_TW.UTF-8 0 0

    UUID=01D98685D8342A50 /media/bitlocker-test ntfs-3g _netdev,defaults,nodev,nosuid,locale=zh_TW.UTF-8 0 0
    UUID=01D98B4612C4F6D0 /media/vhdx-test ntfs-3g _netdev,defaults,nodev,nosuid,locale=zh_TW.UTF-8 0 0
    UUID=01D98B437A03A830 /media/bitlk-test ntfs-3g _netdev,defaults,nodev,nosuid,locale=zh_TW.UTF-8 0 0
    UUID=01D98B437CB35EE0 /media/bitlk-test2 ntfs-3g _netdev,defaults,nodev,nosuid,locale=zh_TW.UTF-8 0 0
    ```
    where the BitLocker partitions would be mapped in `/dev/mapper`. In the example described formerly, they would be `/dev/mapper/bitlk-d3eed7e3-01`, `/dev/mapper/bitlk-bace267f-01` and `/dev/mapper/bitlk-bace284a-01`. The `UUID`s of BitLocker partitions would be of the mapped ones and not of `/dev/nbdXpY`. The forth entry demonstrates a non-BitLocker partition in `bitlk-test.vhd` which is the same disk image that `bitlk-test` was in. Note that the preexistent partition where the disk image file lives must be mounted, of course. Option `_netdev` is required for partitions on NBD, otherwise it will hang on boot; you will also need `X-mount.mkdir` if you want to mount them with `mount -a` (in command prompt, sometimes). 

* BitLocker partition in dynamically expanding disk image on an NTFS volume

  You might encounter a problem that after you modified the files in a BitLocker volume under Linux, the disk image cannot be mounted in Windows 10 anyomre. The error message says `Make sure the file is in an NTFS volume and isn't in a compressed folder or volume.` As far as we know by googling, it seems an issue (or a feature?) of Windows security update KB4019472. There are several ways to workaround this but if the disk image was huge, then run
  ```
  fsutil sparse setFlag <YOUR-DISK-IMAGE-FILENAME> 0
  ```
  in the command prompt would be a reasonable and handy one. 
  
