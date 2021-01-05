# RaspberryPi_NAS
Resources I used to create a home server using a Raspberry Pi 4, an external hard drive, Samba, Plex, and MAC OS.

### Install Raspberry Pi OS
I just went the easy route on this and installed the OS via the Raspberry Pi Imager found on the [Raspberry Pi website](https://www.raspberrypi.org/software/). Download the installation software to your mac, plug in a microSD card, and follow the instructions. 

### Boot Raspberry Pi
I attempted to do this headless (without a monitor attached) but was unsuccessful. I finally caved and bought a little HDMI to micro HDMI adapter to debug my Pi. Apparently, at some point after installing the microSD, I unplugged the Pi without shutting it down properly. This corrupted the boot partition, which was why I couldn't SSH into it. I reformatted the microSD, installed it, and plugged the Pi into an ethernet cable. I followed the OS set-up instructions, enabled SSH (using the GUI), and the installation was successful.    

To properly shut down the Pi: `sudo shutdown`    
To properly restart the Pi: `sudo reboot`   

### Set a Static IP for Running Headless
I followed this [tutorial](https://pimylifeup.com/raspberry-pi-static-ip-address/) to set a local static IP. Initially, I tried an arbitrary IP address, 192.168.1.68, but my router reassigned it to 192.168.0.68. Apparently, my router only allows certain range of addresses. Thus, I changed the IP to begin with 192.168.0.x  (where x is in the range of 2-254) and the static IP finally persisted. I added this code to the end of my dhcpcd.conf file:    
`interface eth0`            
`static ip_address=192.168.0.44/24`          
`static routers=192.168.0.1`            
`static domain_name_servers=68.105.28.11 68.105.29.11 68.105.28.12`    

### SSH into Pi
This was an exciting moment. After all my troubles with installation, I ran this command on my Mac terminal and it finally worked!    
`ssh pi@192.168.0.44`    

SSH is important because I will be running my Pi server headless. It's connected to my router in another room but I still need access to it.

### Format HDD File System
I plugged an old Mac hard drive into my Pi using a powered USB hub and ran the command:   
`sudo lsblk -o UUID,NAME,FSTYPE,SIZE,MOUNTPOINT,LABEL,MODEL`   
         
This command informs you of the drive name on the Pi, in my case, `sda2` (important for mounting) and the FSTYPE of the drive. My drive is hfs+, which is an older Mac specific file system type which will work on the Raspberry Pi but it's not compatible with Windows. The only other file type that is compatible for both read and write between Mac, Pi (Linux), and Windows is exFAT. Unfortunately, Mac's Time Machine software will only work with hfs+, thus I didn't reformat the drive.

### Permanently Mount a Hard Drive with Permissions
I predominantly followed the [Raspberry Pi docs](https://www.raspberrypi.org/documentation/configuration/external-storage.md) but had to change the mount command slightly by using the “-t hfsplus” argument. This flag tells the mount application to use hfsplus to understand the formatting of the drive: `sudo mount -t hfsplus /dev/sda2 /mnt/hd1`. Then, I made the following changes to the fstab file in order to automatically mount the drive every time it's plugged in (or the Pi is restarted). To edit the file, type:     
`sudo nano /etc/fstab`   

Then I added this line at the bottom of the file:   
`UUID=c2e7ea36-74e9-313c-8bc4-3fa527ef122b /mnt/hd1 hfsplus force,auto,user,uid=1000,gid=1000,rw,nofail,x-systemd.device-timeout=30 0 0`  

The `UUID` is the unique identifier of my hard drive.  
`/mnt/hd1` is the location where I mounted the drive and where it should persist.         
`hfsplus` is the file system type.   
`force` helped to fix an error where the drive was mounting as read only.      
`uid=1000,gid=1000` gives the user `pi` permission to make changes to the drive files.   
`rw` allows users read and write access.   
`nofail` and the subsequent code allows the Pi to continue booting without error after only waiting 30 seconds if the drive is detached.   

I restarted my Pi and ran the `sudo lsblk -o UUID,NAME,FSTYPE,SIZE,MOUNTPOINT,LABEL,MODEL` command to ensure the drive mounted properly.   
If you ever want to manually unmount the drive without shutting down the Pi simply type: `sudo umount /mnt/hd1`.

### Install Samba
First I created a shared folder on my newly mounted hard drive: `mkdir /mnt/hd1/shared`. Then, I followed this [tutorial](https://pimylifeup.com/raspberry-pi-samba/). While installing Samba, you will be prompted “Modify smb.conf to use WINS from DHCP?” Press “Enter” to select “No.” I wasn't too sure about this but a couple of websites mentioned most home users won't need this setting.     

I configured my smb.conf file to give users read/write permissions by running the command `sudo nano /etc/samba/smb.conf` and adding this:       
`[share]`       
   `path=/mnt/hd1/shared`             
   `writeable=yes`             
   `read only=no`             
   `create mask=0777`              
   `directory mask=0777`             
   `public=no`      
    
### Connect to the Samba share on Mac
With the “Finder” application open, click the “Go” button in the toolbar, then click the “Connect to Server...” option. Within the address box enter in the details of your Raspberry Pi’s SMB share. In my case: `smb://192.168.0.44/share`. Click connect and then enter your Samba username and password. This will successfully set up a network drive that can be accessed on both a Windows PC and a Mac computer.       


### Automating Backups with Time Machine
I created another folder on the hard drive for storing backups `mkdir /mnt/hd1/backups`. Then I added a new share definition to the smb.conf file by running the command `sudo nano /etc/samba/smb.conf` and adding this:       
`[backups]`       
   `path=/mnt/hd1/backups`             
   `writeable=yes`             
   `read only=no`             
   `create mask=0777`              
   `directory mask=0777`             
   `public=no`      
   `vfs objects = catia fruit streams_xattr`     
   `fruit:time machine = yes`    

Next, in Mac System Preferences, I selected `backups` as my backup disk on RASPBERRYPI.local. I followed the prompts and signed in with my username and password. This resulted in an error that I did not have permission to write or append to the drive. I entered this command to change permissions on the server: `sudo sudo chmod -R 777 /mnt/hd1`. Finally, my Mac backups are now automated using Time Machine.    


### Install Plex
I just followed this [tutorial](https://pimylifeup.com/raspberry-pi-plex-server/) to install Plex on Raspberry Pi. Then I pasted the following into my browser to log in: `192.168.0.44:32400/web/`  
I simply signed in and directed Plex to my server's external hard drive media location. I did get some errors but I typed `sudo apt-get update` in my Pi terminal and that fixed it. Now I can stream my media from any Plex app!


### Server Speed
My server speed seems slow so I installed `iperf3` to test the speed between my network devices. Initial testing revealed an inconsistent range of transfer speeds, anywhere from 10 to 35 MB/sec. This is very slow for a Pi NAS, where good speeds are 75-100 MB/sec. After switching from exFAT to hfs+ file system and upgrading to a CAT6 ethernet cable, the speed test averages hovered right around 25 MB/sec. While this is still slow, there was much less variability between tests. I might be able to increase speeds with a SSD instead of HDD, but I'm okay with the current rate for my simple purposes.    
        
While ssh'd into the pi enter the command `iperf3 -s`.     
In a Mac terminal enter the command `iperf3 -c 192.168.0.44` using the IP address of the server. This can also be tested in reverse using your computer as the server. To get your Mac IP address type the command `ipconfig getifaddr en0` and reverse the client and server commands. 
