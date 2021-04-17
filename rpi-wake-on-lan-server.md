## On RPi (WOL server):
Install the tool to generate the WOL "magic packet":
```
sudo apt-get install etherwake
```

To wake the WOL client you will need to run the tool
with the `$IFACE` set to the RPi network interface
connected to the LAN shared with the WOL client, e.g. `eth0`,
and the `$CLIENT_MAC` set to the MAC address of the WOL client:
```
sudo etherwake -b -i $IFACE $CLIENT_MAC
```
Here `-b` is for broadcast.
For some reason, without it the "magic packet" did not come through to the WOL client
(as witnessed by a wireshark packet capture).

## On the desktop (WOL client):
Reboot into BIOS and enable "wake on PCI" or similar.
Check "Supports Wake-on" in the output of:
```
sudo ethtool enp39s0
```
e.g.: "Supports Wake-on: pumbg"

Enable wake-on-lan temporarily (until reboot):
```
sudo ethtool -s enp39s0 wol g
```
Enable wake-on-lan permanently using systemd (from https://askubuntu.com/a/892083/143694):
Create `/etc/systemd/system/wol@.service`
```
[Unit]
Description=Configure Wake On LAN for %i
Requires=network.target
After=network.target

[Service]
Type=oneshot
ExecStart=/sbin/ethtool -s %i wol g

[Install]
WantedBy=basic.target
```

Then register the service for startup (`$IFACE` is your LAN interface on the desktop, e.g. `eth0` or `enp39s0`):
```
sudo systemctl enable wol@$IFACE
```
