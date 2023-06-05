# nbx-vhdx
A systemd service connects vhd/vhdx as NBDs and BitLocker partitions on startup

  # Prerequisites: 

  This tool utilizes `qemu-nbd` for connecting disk image as NBDs and `cryptsetup` for operations on BitLocker partitions, they are required. `blkid` plays an important role in the script, it is also required. The rest commands, for example `mount/unount` are likely pre-installed in all distros. 

  # install/uninstall: 

  Run `install-sh` or `uninstall-sh` along with `nbd-vhdx` and `nbd-vhdx.service` to install/uninstall the service and shell script executable. 
