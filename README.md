# Setup Cyberghost VPN on OpenWRT

## Step 1 - Open VPN installation
1. SSH into the your router
2. Update OPKG Packages & install OpenVPN:
```
opkg update
opkg install openvpn-openssl
```
3. Autostart Open VPN:
```
/etc/init.d/openvpn enable
```
## Step 2 - Cyberghost account
1. Configure a new device
2. Select Open VPN Protocol (UDP)
3. Save configuration file and take note of user and password
4. Rename OVPN file fo CG_XX.conf (Where XX corresponds to the two digit code for the selected country)
5. Create a `user.txt` file an introduce your credentials like this
```
USER
PASSWORD
```
6. Edit CG_XX.conf and change the line
```
[...]
auth-user-pass
[...]
```
to
```
[...]
auth-user-pass /etc/openvpn/user.txt
[...]
```
And don't forget to save the file
6. Copy all the config files to `/etc/openvpn` folder in your router via SCP.

## Step 3 - Configure VPN Client connection
1. Run UCI commands to configure as VPN Client:
```
# a new OpenVPN instance:
uci set openvpn.cyberghost=openvpn
uci set openvpn.cyberghost.enabled='1'
## rename CG_XX.conf accordingly to Step 2
uci set openvpn.cyberghost.config='/etc/openvpn/CG_XX.conf'

# a new network interface for tun:
uci set network.cyberghostvpn=interface
uci set network.cyberghostvpn.proto='none' #dhcp #none
uci set network.cyberghostvpn.ifname='tun0'

# a new firewall zone (for VPN):
uci add firewall zone
uci set firewall.@zone[-1].name='vpn'
uci set firewall.@zone[-1].input='REJECT'
uci set firewall.@zone[-1].output='ACCEPT'
uci set firewall.@zone[-1].forward='REJECT'
uci set firewall.@zone[-1].masq='1'
uci set firewall.@zone[-1].mtu_fix='1'
uci add_list firewall.@zone[-1].network='cyberghostvpn'

# enable forwarding from LAN to VPN:
uci add firewall forwarding
uci set firewall.@forwarding[-1].src='lan'
uci set firewall.@forwarding[-1].dest='vpn'

# Finally, you should commit UCI changes:
uci commit
```

## Step 4 - Configure DNS
1. Run the following UCI commands
```
uci set network.wan.peerdns='0'
uci del network.wan.dns
uci add_list network.wan.dns='10.101.0.243'
uci add_list network.wan.dns='38.132.106.139'
uci commit
```

## Step 5 - Prevent DNS Leaks
1. Edit the file `/etc/firewall.user`
```
# This file is interpreted as shell script.
# Put your custom iptables rules here, they will
# be executed with each firewall (re-)start.

# Internal uci firewall chains are flushed and recreated on reload, so
# put custom rules into the root chains e.g. INPUT or FORWARD or into the
# special user chains, e.g. input_wan_rule or postrouting_lan_rule.
if (! ip a s tun0 up) && (! iptables -C forwarding_rule -j REJECT); then
        iptables -I forwarding_rule -j REJECT
fi
if (! iptables -C forwarding_lan_rule ! -o tun+ -j REJECT); then
        iptables -I forwarding_lan_rule ! -o tun+ -j REJECT
fi
```
2. Create the file `99-prevent-leak` in the folder `/etc/hotplug.d/iface/` with the following content:
```
#!/bin/sh
if [ "$ACTION" = ifup ] && (ip a s tun0 up) && (iptables -C forwarding_rule -j REJECT); then
        iptables -D forwarding_rule -j REJECT
fi
if [ "$ACTION" = ifdown ] && (! ip a s tun0 up) && (! iptables -C forwarding_rule -j REJECT); then
        iptables -I forwarding_rule -j REJECT
fi
```

## Step 6 - Connect
1. Since we modified firewall we need to run
```
/etc/init.d/firewall reload
```
2. Since we added a new interface we need to restart network daemon (you will lose connectivity for a moment)
```
/etc/init.d/network restart
```
3. Start Open VPN
```
/etc/init.d/openvpn start
```

## Test VPN Connection
1. Test Your IP Address
```
https://whatismyipaddress.com/
```
2. Run a DNS Leak Test
```
https://dnsleaktest.com/
```
