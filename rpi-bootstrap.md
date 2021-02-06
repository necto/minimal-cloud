- initialize the SD card: 
```
wget https://downloads.raspberrypi.org/raspios_lite_armhf/images/raspios_lite_armhf-2021-01-12/2021-01-11-raspios-buster-armhf-lite.zip -q -O -| funzip | sudo dd of=/dev/mmcblk0 bs=4M conv=fsync status=progress
```

- enable sshd: (using `sudo raspi-config` -> Interface options -> SSH) https://www.raspberrypi.org/documentation/remote-access/ssh/
- Now I can disconnect the keyboard and the TV and ssh from my machine.
- change password for the `pi` user: `passwd`
- copy over the public key: `cat > .ssh/authrozed_keys`; `chmod 700 .ssh/authorized_keys`

```
# Generate locale
sudo locale-gen en_US.UTF-8
sudo dpkg-reconfigure locales

# Setup Docker
sudo apt update && sudo apt upgrade
sudo apt install docker-compose
sudo usermod -aG docker pi
```

- Reconnect to get the `docker` group
