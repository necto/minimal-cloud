# Initial setup
## Initialize the SD card: 
The following command downloads a light version of raspios, and flushes it into the sdcard (/dev/mmcblk0):
```
wget https://downloads.raspberrypi.org/raspios_lite_armhf/images/raspios_lite_armhf-2021-01-12/2021-01-11-raspios-buster-armhf-lite.zip -q -O -| funzip | sudo dd of=/dev/mmcblk0 bs=4M conv=fsync status=progress
```
Takes 4.5 minutes

## Enable SSHd on rpi
### Headless
mount the boot partition
```
mkdir -p sd
sudo mount /dev/mmcblk0p1 sd
```
Add a file named `ssh`:
```
sudo touch sd/ssh
echo "enable SSHd" > /media/necto/boot/ssh
```
Unmount it
```
sudo umount sd
rmdir sd
```
Insert the sdcard into rpi, connect it to the network and boot.

### Interactive
- insert the sdcard into rpi, connect keyboard and display, boot it
- using `sudo raspi-config` -> Interface options -> SSH (see https://www.raspberrypi.org/documentation/remote-access/ssh/)
- Now disconnect the keyboard and the TV

### Secure SSHd
Rpi should be booted from the sdcard and reachable by ssh.
- ssh into rpi with user `pi` and password `raspberry`
- copy over the public key:
```
mkdir ~/.ssh
chmod 700 .ssh
cat > .ssh/authrized_keys
chmod 600 .ssh/authorized_keys
```
- reconnect to make sure the public-key auth works
- change password for the `pi` user: `passwd`

## Generate locale
```
sudo locale-gen en_US.UTF-8
sudo dpkg-reconfigure locales
```


# Install the tools
Install the packages:
- `syncthing` - to synchronize files with other machines
- `nginx` (with `nginx-extras` and `apache2-utils`) - to serve the files via WebDAV
- `certbot` - to enable HTTPS with let's encrypt
- `tmux` - just for convenient remote experience
```
sudo apt update
sudo apt install syncthing nginx nginx-extras apache2-utils certbot python3-certbot-nginx
```

## Setup Syncthing
Start syncthing:
```
syncthing -gui-address=192.168.1.18:5354
```
Connect with your webbrowser to `192.168.1.18:5354` and click on the "options" gear,
then the "GUI" tab.
Enter `192.168.1.18:5354` to the gui address box, and user and a password for the configuration access.

Make sure the `syncthing@pi.service` is configured. On raspbian it should be by default.
See https://docs.syncthing.net/users/autostart.html#linux

## Setup WebDAV

copy nginx.conf to /etc/nginx/nginx.conf
```
sudo htpasswd -cb /etc/nginx/webdavpasswd $WEBDAV_USERNAME $WEBDAV_PASSWORD
#sudo mkdir /media/data/
#sudo chmod -R o+w /media/data/
```

Reload the nginx configuration:
```
sudo nginx -s reload
```

## Setup Organice

### Install Node.js
```
wget https://unofficial-builds.nodejs.org/download/release/v12.18.3/node-v12.18.3-linux-armv6l.tar.gz
tar -xzf node-v12.18.3-linux-armv6l.tar.gz
sudo cp -R node-v12.18.3-linux-armv6l/* /usr/local
rm node-v12.18.3-linux* -r
```
### Install forever
Install `forever` to be able to daemonize the organice web server:
```
sudo npm install --global forever forever-service
```
### Yarn
```
sudo npm install --global yarn
sudo yarn global add serve
```

### Build
run `yarn build` on your host (or in a docker container), copy the build directory to rpi:organice/

## Setup the IPv6
To circumvent the lack of a public IPv4, if you have a public IPv6, you can use http://v4-frontend.netiter.com/

For that you need to bind a particular static IPv6 that is specified in the AAAA record for you domain name.

Add the static IPv6 in `/etc/dhcpcd.conf`, and reload the configuration:
```
echo "interface eth0" | sudo tee -a /etc/dhcpcd.conf
echo "static ip6_address=2a04:ee41:82:72d7:937e:f7d:56ae:f46a" | sudo tee -a /etc/dhcpcd.conf
sudo systemctl daemon-reload
sudo systemctl restart dhcpcd
```

## Setup HTTPS
```
sudo certbot --nginx
```
Answer the interactive questions:
* enter your e-mail,
* agree with ToS,
* choose to share or not your email with EFF,
* choose the domain name to get the certificate for,
* choose whether or not to redirect HTTP to HTTPS

Check if certificate renewal works:
```
sudo certbot renew --dry-run
```

# Run the server
The following creates the organice service and then for all three services:
- `organice` (just created),
- `syncthing@pi.service` (provided by the syncthing package)
- `nginx` (provided by the nginx package)
it makes sure they run forever:
- `enable` to enable restarting the service after system restarts
- `start` to launch the service now

```
sudo forever-service install organice --start --script /usr/local/bin/serve --scriptOptions " -s /home/pi/organice"
sudo systemctl enable organice
sudo systemctl start organice
sudo systemctl enable syncthing@pi.service
sudo systemctl start syncthing@pi.service
sudo nginx -s reload
sudo systemctl enable nginx
sudo systemctl start nginx
```

# Backup the SD card
```
sudo apt-get clean
sudo shutdown now
```
Remove the card from raspberrypi and insert it into the cardreader.
Create the image (takes under 3 minutes):
```
sudo dd bs=4M if=/dev/mmcblk0 of=rpi.img conv=fsync status=progress
sudo chown $USER: rpi.img
```
Shrink the image:
```
wget https://raw.githubusercontent.com/necto/PiShrink/master/pishrink.sh
chmod +x pishrink.sh
sudo ./pishrink.sh -aZ rpi.img rpi.shrunk.img
```

xz compression (-Z): 10 minutes, shrunk the 7.5G image to 732M
gz compression (-z): ~4 minutes, shrunk to 1022M

## Restore the SD card
```
unxz rpi.shrunk.img.xz
sudo dd bs=4M if=rpi.shrunk.img of=/dev/mmcblk0 conv=fsync status=progress
```
Takes 8 minutes
