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
echo "enable SSHd" > /media/necto/boot/ssh
```
### Interactive
- insert the sdcard into rpi, connect keyboard and display, boot it
- using `sudo raspi-config` -> Interface options -> SSH (see https://www.raspberrypi.org/documentation/remote-access/ssh/)
- Now disconnect the keyboard and the TV
### Secure SSHd

- ssh into rpi with user `pi` and password `raspberrypi`
- change password for the `pi` user: `passwd`
- copy over the public key: `cat > .ssh/authrozed_keys`; `chmod 700 .ssh/authorized_keys`

# Install the tools
## Generate locale
```
sudo locale-gen en_US.UTF-8
sudo dpkg-reconfigure locales
```

## Setup syncthing
```
sudo apt install syncthing
syncthing -gui-address=192.168.1.18:5354
```

## Setup WebDAV

```
sudo apt install nginx nginx-extras apache2-utils
```
copy webdav-organice/nginx.conf to /etc/nginx/nginx.conf
remove the `daemon off;` directive (it is not needed without docker)
change the port number from 8080 to 80
add `listen [::]:80 default server` on the next line
replace `server_name localhost` with `server_name zaostrovykh.ch`
```
sudo mkdir /media/data/
sudo htpasswd -cb /etc/nginx/webdavpasswd $WEBDAV_USERNAME $WEBDAV_PASSWORD
sudo chmod -R o+w /media/data/
```

## Setup Organice

### Node.js
wget https://unofficial-builds.nodejs.org/download/release/v12.18.3/node-v12.18.3-linux-armv6l.tar.gz
tar -xzf node-v12.18.3-linux-armv6l.tar.gz
cd node-v12.18.3-linux-armv6l
sudo cp -R * /usr/local
cd ..
rm node-v12.18.3-linux* -r

### Yarn
sudo npm install --global yarn

### Build
```
git clone https://github.com/necto/organice
cd organice
yarn config set network-timeout 300000 # Workaround for the slow SDCard (date-fns-2.16.1.tgz: ESOCKETTIMEDOUT)

yarn install # slow (takes 2021.55s)
sudo yarn global add serve
# yarn build # runs out of memory
# yarn cache clean --- probably not necessary
# rm -rf node_modules --- probably not necessary
```

run `yarn build` on your host (or in a docker container), copy the build directory to rpi:organice/

```
serve -s ~/organice/build
```

## Setup the ip

in /etc/dhcpcd.conf:
interface eth0
static ip6_address=2a04:ee41:82:72d7:937e:f7d:56ae:f46a

then

sudo systemctl daemon-reload
sudo systemctl restart dhcpcd

## Setup HTTPS
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx

## Backup the SD card
sudo apt-get clean
sudo shutdown now

Remove the card from raspberrypi and insert it into the cardreader
Create the image (takes under 3 minutes)
sudo dd bs=4M if=/dev/mmcblk0 of=rpi.img conv=fsync status=progress
sudo chown $USER: rpi.img

wget https://raw.githubusercontent.com/necto/PiShrink/master/pishrink.sh
chmod +x pishrink.sh
sudo ./pishrink.sh -aZ rpi.img rpi.shrunk.img

xz compression (-Z): 10 minutes, shrunk the 7.5G image to 732M
gz compression (-z): ~4 minutes, shrunk to 1022M

## Restore the SD card
unxz rpi.shrunk.img.xz

sudo dd bs=4M if=rpi.shrunk.img of=/dev/mmcblk0 conv=fsync status=progress
Takes 8 minutes
