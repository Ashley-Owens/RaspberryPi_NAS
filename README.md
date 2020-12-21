# RaspberryPi_NAS
How I set up my home server (in case I forget!) using a Raspberry Pi 4, an external hard drive, Samba, and MAC OS.

### Setting up Raspberry Pi OS
I just went the easy route on this and installed the OS via the Raspberry Pi Imager found on the [Raspberry Pi website](https://www.raspberrypi.org/software/). Download the installation software to your mac, plug in a microSD card, and follow the instructions. 

### Booting Raspberry Pi
I attempted to do this headless (without a monitor attached) but was unsuccessful. I finally caved and bought a little HDMI to micro HDMI adapter to debug my Pi. Apparently, at some point after installing the microSD, I unplugged the Pi without shutting it down properly. This corrupted the boot partition, which was why I couldn't SSH into it. I reformatted the microSD, installed it, and plugged the Pi into an ethernet cable. I followed the OS set-up instructions, enabled SSH (using the GUI), and the installation was successful.

### Raspberry Pi Static IP for Running Headless
I followed this [tutorial](https://pimylifeup.com/raspberry-pi-static-ip-address/) to set my local static IP. I tried a few different IP addresses, apparently my router only allows IP's in the range of 192.168.0.2-254. Initially, I tried an IP address outside of this range, 192.168.1.68, but when I checked my router it had assigned 192.168.0.68 to my Pi, which was why I couldn't SSH into it. I changed the IP to begin with 192.168.0.x and the static IP finally persisted.

### SSH into Pi
This was an exciting moment. After all my troubles with installation, I ran this command on my Mac terminal and it finally worked!    
`ssh pi@192.168.0.x`    

SSH is important because I will be running my Pi server headless. It's connected to my router in another room but I still need access to it.

### Format HDD File System
I plugged an old Mac hard drive into my Pi using a powered USB hub and ran the command:   
`sudo lsblk -o UUID,NAME,FSTYPE,SIZE,MOUNTPOINT,LABEL,MODEL`   
         
The FSTYPE of my drive was hfs+. This is a Mac specific file system type which will work on the Raspberry Pi but I couldn't find much documentation on it. The only other file type that is compatible for both read and write between Mac and Pi (Linux) is exFAT. The drive wasn't mounted yet so I unplugged it and reformatted it using my Mac. I changed the file type to exFAT, which also wiped the drive. I plugged it back into the Pi, ran the command again, and now the FSTYPE was listed as `exfat`. This command also informs you of the drive name on the Pi, in my case, `sda2` (important for mounting).   

### Mount the Hard Drive
As exFAT isn't native to Pi, it needs a package to support the file system format. I followed two tutorials. From this [tutorial](https://pimylifeup.com/raspberry-pi-exfat/), I only used step two for mounting the drive. I predominantly followed the [Raspberry Pi docs](https://www.raspberrypi.org/documentation/configuration/external-storage.md) for everything else. In order for the drive to automatically mount every time it's plugged in (or the Pi is restarted), I made the following changes to the fstab file (this just has some minor alterations from the tutorials). To edit the file, type:     
`sudo nano /etc/fstab`   

Then I added this line at the bottom of the file:   
`UUID=5FE0-DD85 /mnt/hd1 exfat defaults,auto,user,uid=1000,gid=1000,rw,nofail,x-systemd.device-timeout=30 0 0`  

The `UUID` is the unique identifier of my hard drive.  
`/mnt/hd1` is the location where I mounted the drive and where it should persist.         
`exfat` is the file system type.   
`uid=1000,gid=1000` gives the user `pi` permission to make changes to the drive.   
`rw` allows users read and write access.   
`nofail` and the subsequent code allows the Pi to continue booting without error after only waiting 30 seconds if the drive is unattached.   

I restarted my Pi and ran the `sudo lsblk -o UUID,NAME,FSTYPE,SIZE,MOUNTPOINT,LABEL,MODEL` command to ensure the drive mounted properly. If you ever want to manually unmount the drive without shutting down the Pi simply type: `sudo umount /mnt/hd1`.

### Set Up Samba

