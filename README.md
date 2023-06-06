# nbx-vhdx
nbx-vhdx is a systemd service that connects VHD/VHDX files as NBDs (Network Block Devices) and handles BitLocker partitions during system startup.

* Prerequisites: 

  To use this tool, you will need the following dependencies: `qemu-nbd` for connecting disk images as NBDs, `cryptsetup` (version 2.6.1) for working with BitLocker partitions, and `blkid`, which plays a crucial role in the script. The remaining commands, such as mount/umount, are typically pre-installed in most Linux distributions.

* Installation/Uninstallation: 

  To install or uninstall the service and the shell script executable, run the `install-sh` or `uninstall-sh` scripts along with `nbd-vhdx` and `nbd-vhdx.service`.
  
* Command Line Usage: 

  Run `nbd-vhdx` without any arguments to see a list of valid commands.
  
* /etc/vhdxtab

  The `/etc/vhdxtab` file is used to describe disk images and BitLocker partitions that are set up during system boot. The format of this file is based on `/etc/crypttab`, with an additional field introduced for specifying the filename of the disk image. Here are some important points to note:

  1. The first field (`target name`) is ignored as BitLocker partitions will be mapped as `/dev/mapper/bitlk-<uuid>`, where <uuid> represents the actual partition UUID. User-defined names are not required or supported.

  2. The fourth field (`options`) is ignored, as it only accepts the `bitlk` type and does not support any additional options.

  3. If you only want to connect a disk image as an NBD without mapping any BitLocker partitions, leave the first four fields as "-".

  4. If there are multiple BitLocker partitions within the same disk image, add each entry with the same filename. This means that the same filename can appear in multiple entries, and it will be connected as a single NBD.

  Here's an example: 
  ```
  # <target name>	<source device>		<key file>	<options>
  - PARTUUID=d3eed7e3-01 /root/d3eed7e3-01.fvek bitlk /media/bin/bitlocker-test.vhdx
  - PARTUUID=bace267f-01 /root/bace267f-01.fvek bitlk /media/bin/vhdx-test.vhdx
  - - - - /media/bin/bitlk-test.vhd
  - PARTUUID=bace284a-01 /root/bace284a-01.fvek bitlk /media/bin/bitlk-test.vhd
  ```
  The third and fourth entries suggested the same file is redundant. However, if you want to pass the entire device through to a virtual machine connected to `bitlk-test.vhd` without mapping any BitLocker partition within it, you can simply comment out the fourth entry. On the other hand, if you have non-BitLocker partitions in `bitlk-test.vhd` that you want to mount and describe using `/etc/fstab`, you can choose either the third entry (to not map the BitLocker partition) or the fourth entry (to mount them all together). You can leave them as they are or comment them out based on your requirements.

* /etc/fstab

    In the `/etc/fstab` file, you can describe partitions that need to be mounted at startup as usual. Here's an example:
    
    ```
    UUID=xxxxxxxxxxxxxxxx /media/bin ntfs-3g defaults,nodev,nosuid,locale=zh_TW.UTF-8 0 0

    UUID=01D98685D8342A50 /media/bitlocker-test ntfs-3g _netdev,defaults,nodev,nosuid,locale=zh_TW.UTF-8 0 0
    UUID=01D98B4612C4F6D0 /media/vhdx-test ntfs-3g _netdev,defaults,nodev,nosuid,locale=zh_TW.UTF-8 0 0
    UUID=01D98B437A03A830 /media/bitlk-test ntfs-3g _netdev,defaults,nodev,nosuid,locale=zh_TW.UTF-8 0 0
    UUID=01D98B437CB35EE0 /media/bitlk-test2 ntfs-3g _netdev,defaults,nodev,nosuid,locale=zh_TW.UTF-8 0 0
    ```
    The BitLocker partitions will be mapped under `/dev/mapper`. In the example mentioned earlier, they would be mapped as `/dev/mapper/bitlk-d3eed7e3-01`, `/dev/mapper/bitlk-bace267f-01`, and `/dev/mapper/bitlk-bace284a-01`. The UUIDs of the BitLocker partitions will be those of the mapped devices and not `/dev/nbdXpY`. The fourth entry demonstrates a non-BitLocker partition in `bitlk-test.vhd` which is the same disk image mentioned in the `bitlk-test` entry. Note that the pre-existing partition where the disk image file resides must be mounted. Additionally, the `_netdev` option is required for the partitions on NBD; otherwise, the system may hang during boot. If you want to mount them with `mount -a` in the command prompt, you will also need `X-mount.mkdir`.

* BitLocker Partition in a Dynamically Expanding Disk Image on an NTFS Volume
  
  You may encounter a problem where, after modifying files in a BitLocker volume under Linux, the disk image cannot be mounted in Windows 10 anymore. The error message "Make sure the file is in an NTFS volume and isn't in a compressed folder or volume." may appear. Based on my research, this issue seems to be related to Windows security update KB4019472. There are several workarounds, but if the disk image is large, running the following command in the command prompt would be a reasonable and handy one:

  ```
  fsutil sparse setFlag <YOUR-DISK-IMAGE-FILENAME> 0
  ```
