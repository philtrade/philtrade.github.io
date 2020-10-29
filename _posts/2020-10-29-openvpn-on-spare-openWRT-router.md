# Reusing Spare Router - roll your own VPN server with `openWRT`

Got a spare router?  `openWRT` can turn it into a swiss army knife for home networking.

Previously, on a spare router flashed with `openWRT`, I implemented a wake-over-WAN mechanism.  It saved me lot of time in remotely waking up a home GPU dev box than spinning up a commercial cloud instance.

Lately I was asked about private/non-commercial VPN solution to masquerade outbound traffic.  I followed, tweaked, and consolidated the instructions from multiple sources into a single reusable script ([the gist is here](https://gist.github.com/philtrade/88bf4168b33b35b04667c5d56bfbfd10)).

The exercise helped me understand some practical tradeoffs in crypto key generation and management, and configuring network services using `uci` vs direct commands.


### — Technicalities —

The original instruction by [`openWRT` VPN server wiki page](https://openwrt.org/docs/guide-user/services/vpn/openvpn/server) has a major bottleneck in generating a 2048-bit Diffie-Hellman parameter file (aka`dh.pem`): it can take 45 minutes to hours on the router.  On an Intel E5 xeon it would only take 25 seconds.

A couple of workarounds which allows generating a 4096 bit `dh.pem` bearable:

1. By calling [‘`openssl dhparam`’ with ‘`-dsaparam`’ option](https://security.stackexchange.com/questions/95178/diffie-hellman-parameters-still-calculating-after-24-hours).  Generating a 4096-bit `dh.pem` now only takes a 2-3 minutes on the router, and only 5 seconds on the Intel E5 Xeon (10 seconds for 8192-bit).
2. Let user provide a dh.pem, which can be generated elsewhere, and supply it to the VPN setup script.

The original instruction does not allow WAN access from behind the VPN. It can be solved by an iptables `SNAT` rule, e.g. [`iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -j SNAT --to-source <router ip>`](http://dani.foroselectronica.es/openvpn-openwrt-secure-browsing-from-your-mobile-phone-283/)

I opted for generating separate firewall rules file and server instance config, then "include" them using `uci`.  The advantage is those can be easily tweaked/modified by editing the files with text editor, and that it avoids polluting the main `/etc/config/{firewall.user,openvpn}` files.  The disadvantage is that `uci show` or `luci` the web interface will not show the details.
