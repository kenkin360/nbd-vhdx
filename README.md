# nbd-vhdx
nbd-vhdx is a systemd service that connects VHD/VHDX files as NBDs (Network Block Devices) and handles BitLocker partitions during system startup.

* **Prerequisites**: 

  You'll need to load `nbd` kernel module at boot time. Pick an existing `.conf` file which is located in `/etc/modules-load.d` or just create a new one such like `/etc/modules-load.d/nbd.conf` and add the following lines: 

	  nbd
	  options nbd max_part=255

   `max_part` is the number of partitions per device, it defaults to `0` that causes problems. You may choose a value below 255 depend on your needs. 

  To get `nbx-vhdx` work, you will also need the following dependencies:

    * `qemu-nbd` for connecting disk images as NBDs
    * `cryptsetup` (version 2.6.1) for dealing with BitLocker partitions
    * `blkid` which plays a crucial role in my scripts

  For Debian-based Linux distros users, run the following command to install necessary packages:

      apt install qemu-utils util-linux

* **Installation/Uninstallation**: 

  To install or uninstall the service and the shell script executable, run the `install-sh` or `uninstall-sh` scripts along with `nbd-vhdx` and `nbd-vhdx.service`.
  
* **Command Line Usage**: 

  Run `nbd-vhdx` without any arguments to see a list of valid commands.
  
* **/etc/vhdxtab**

  The file `/etc/vhdxtab` is for disk image and BitLocker partition configurations that are set up at system boot.

  The format of this file is based on `/etc/crypttab`, and an additional field for the filename of the disk image is introduced.

  Some explaination of the format worthy to note come to mind:

  1. The 1st field (target name) is ignored as BitLocker partitions will be mapped like `/dev/mapper/bitlk-<uuid>`, where `<uuid>` represents the actual partition UUID. User-defined names are not required and unsupported.

  2. The 4th field (options) is ignored as it accepts only `bitlk` and not any options else.

  3. If all you want is just to connect a vhdx to NBD(without mapping any BitLocker partitions), then leave the first four fields like `- - - -`.

  4. If there are multiple BitLocker partitions within the same disk image, add each entry with the same filename. This means that the same filename can appear in multiple entries, and it will be connected as a single NBD.

  Here's an example: 
  ```
  # <target name>	<source device>		<key file>	<options>
  
  # Simply connect vhdx to NBD
  - - - - /media/ST4000DM001-1ABCD4/games-vhd/391220.vhdx
  
  # Connect a block device to NBD
  - - - - /dev/disk/by-id/ata-ST5000LM007-1ABCD4_WY2ABCD2
  
  # Connect BitLocker partition using PARTUUID with a Full Volume Encryption Key
  - PARTUUID=0dabcd78-01 /root/0dabcd78-01.fvek bitlk /dev/disk/by-id/ata-ST9000LM001-1R8174_WY2ABCD2
  ```

  `nbd-vhx` will try to connect 'bitlk' type partitions using `cryptsetup` automatically if FVEK is provided. In that case, the encrypted partition is exposed as unlocked to your systemm, so that you can use `/etc/fstab` to mount it at system startup. 

  You'll be prompted for password when you try to access a Bitlocker partition on the connected NBD if FVEK wasn't provided. 

* **/etc/fstab**

    Use `blkid` to figure out the UUID or PARTUUID after you connect the disk image or block device to NBDs. Partitions on a connected NBD will be shown like `/dev/nbdXpY` in the output of `blkid`, you may add the entries for them to `/etc/fstab` as usual, except the Bitlocker encrypted partions. 

    The unlocked BitLocker partitions will be mapped in `/dev/mapper`, for the example shown above, it would look like:

      /dev/mapper/bitlk-0dabcd78-01: LABEL="ST9000LM001-1R8174_WY2ABCD2" BLOCK_SIZE="512" UUID="01ABCDBD29DABCD0" TYPE="ntfs"

  in the output of `blkid`. And you may add the line to `/etc/fstab` to make it mounted at startup:

      UUID=01ABCDBD29DABCD0 /media/ST9000LM001-1R8174_WY2ABCD2 ntfs-3g _netdev,defaults,nodev,nosuid,locale=zh_TW.UTF-8 0 0

  Note that the partition where the a vhdx resides must be mounted before you can connect the disk image. Additionally, the `_netdev` option is required for the partitions on NBD; otherwise, the system may hang during boot. As a sidenote, if you want to mount them with `mount -a` in the command prompt, you will also need `X-mount.mkdir`.

* **Some technical detials**

  * Persistent Naming
  
    nbd-vhdx will create the symbolic link of the connected device in `/dev/disk/by-id/nbd-<PTUUID>`, where `<PTUUID>` is the partition table UUID. 
  
  * Bitlocker volume of a dynamically-expanding VHDX file in NTFS

   You might see the error message "Make sure the file is in an NTFS volume and isn't in a compressed folder or volume." after modifying files of a BitLocker volume on Linux and the vhdx can no longer be mounted on Windows. 
    
     Try the following command to workarounds this:
  
    ```
    fsutil sparse setFlag <YOUR-DISK-IMAGE-FILENAME> 0
    ```
