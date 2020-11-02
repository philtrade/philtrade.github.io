# Reusing Spare Router - roll your own VPN server with `openWRT`

Got a spare router?  `openWRT` can turn it into a swiss army knife for home networking.

Previously, on a spare router flashed with `openWRT`, I implemented a wake-over-WAN mechanism.  It saved me lot of time in remotely waking up a home GPU dev box than spinning up a commercial cloud instance.

Lately I was asked about private/non-commercial VPN solution to masquerade outbound traffic.  I followed, tweaked, and consolidated the instructions from multiple sources to get `openvpn` server running on the inquirer's spare router. I further consolidated the steps [into a single script](https://gist.github.com/philtrade/88bf4168b33b35b04667c5d56bfbfd10).

The exercise helped me understand some practical tradeoffs in crypto key generation and management, and configuring network services using `uci` vs direct commands.


### — Technicalities —

#### Make sure you can connect to the spare router after flashing `openWRT`
By default, the new `openWRT` image would disable the router's wifi, and is only accessible via the LAN interface/port as `192.168.1.1`.  If you are using a laptop to configure, make sure it can take a LAN cable.  I didn't have the cable adaptor, and had to connect via another wifi router configured as a DHCP client of the target spare router.

#### Speeding up `dh.pem` creation
The original instruction by [`openWRT` VPN server wiki page](https://openwrt.org/docs/guide-user/services/vpn/openvpn/server) has a major bottleneck: generating a 2048-bit Diffie-Hellman parameter file (aka`dh.pem`) can take 45 minutes to hours on the router (an Intel E5 xeon only takes 25 seconds.)  Two workarounds:

1. By calling [‘`openssl dhparam`’ with ‘`-dsaparam`’ option](https://security.stackexchange.com/questions/95178/diffie-hellman-parameters-still-calculating-after-24-hours).  Generating a twice as long, 4096-bit `dh.pem` now only takes a 2-3 minutes on the router, and only 5 seconds on the Intel E5 Xeon.

2. Generate the `dh.pem` elsewhere on a faster machine, and supply it to [the VPN setup script](https://gist.github.com/philtrade/88bf4168b33b35b04667c5d56bfbfd10).

#### Routing and Firewall Rules, and VPN config file
Opening of WAN port for incoming VPN is done by `uci` commands.  Traffic routing and openvpn parameters are stored in files in `/etc/openvpn/{firewall,server}*`.  The downside is that `uci show`/`luci` cannot display the details for you, they only show the included file names.

Routing outbound traffic through the upstream WAN interace is done by a `iptables` `SNAT` rule in `/etc/openvpn/firewall.ovpn_<port>`, e.g. `iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -j SNAT --to-source <router ip>`, [as explained here](http://dani.foroselectronica.es/openvpn-openwrt-secure-browsing-from-your-mobile-phone-283/)

