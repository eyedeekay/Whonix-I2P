### This Guide is still WIP
If you find errors or problems please post them on the Whonix Forum Post.
See https://forums.whonix.org/t/i2p-integration/4981 for more Informations 
#### If you encounter any problem, post your log with your post (see Debug)

### Preparation

**Create a separate Gateway (TemplateVM&) ProxyVm and Workstation (TemplateVM&) AppVM
Installing I2P**
### Whonix Gateway (Template)VM


**We’ll install I2P using the Debian-testing packages.**

**Add the Debian-testing(buster) repo to your APT list**

`sudo su -c "echo -e 'deb tor+http://vwakviie2ienjx6t.onion/debian buster main' > /etc/apt/sources.list.d/testing.list"`

**Update Packages**

`sudo apt-get update`

**Install I2P, its dependencies and (optional) iceweasel:**

`sudo apt-get -t buster install i2p`

**Configure I2P as a service that automatically runs when your system boots, set the amount of Ram to your needs and leave the User as i2psvc**

`sudo dpkg-reconfigure i2p`

**Make the I2P folders persistent in the ProxyVM by adding the following to** `/usr/lib/qubes-bind-dirs.d/40_qubes-whonix.conf`

```
binds+=( '/etc/i2p' )
binds+=( '/var/lib/i2p/i2p-config' )
```

### Whonix Gateway (Proxy)VM

**Edit the firewall rules**

**Add the settings to the Whonix-Firewall User config** `/etc/whonix_firewall.d/50_user.conf` **(Uncomment any SocksPort you dont need)**

```
NO_NAT_USERS+=" $(id -u i2psvc)"
SOCKS_PORT_I2P_BOB=2827
SOCKS_PORT_I2P_TAHOE=3456
SOCKS_PORT_I2P_WWW=4444
SOCKS_PORT_I2P_WWW2=4445
SOCKS_PORT_I2P_IRC=6668
SOCKS_PORT_I2P_XMPP=7622
SOCKS_PORT_I2P_CONTROL=7650
SOCKS_PORT_I2P_SOCKSIRC=7651
SOCKS_PORT_I2P_SOCKS=7652
SOCKS_PORT_I2P_I2CP=7654
SOCKS_PORT_I2P_SAM=7656
SOCKS_PORT_I2P_EEP=7658
SOCKS_PORT_I2P_SMTP=7659
SOCKS_PORT_I2P_POP=7660
SOCKS_PORT_I2P_BOTESMTP=7661
SOCKS_PORT_I2P_BOTEIMAP=7662
SOCKS_PORT_I2P_MTN=8998
```

**Now reload the Whonix Firewall**

`sudo /usr/bin/whonix_firewall`

**Add these Lines to** `/var/lib/i2p/i2p-config/router.config` **to Reseed via Tor, disable Logs and UPNP**

```
stat.full=false
stat.logFile=stats.log
stat.logFilters=
stat.summaries=
i2np.upnp.enable=false
router.reseedProxy.authEnable=false
router.reseedProxyEnable=false
router.reseedSSLDisable=false
router.reseedSSLProxy.authEnable=false
router.reseedSSLProxyEnable=true
router.reseedSSLProxyHost=127.0.0.1
router.reseedSSLProxyPort=9050
router.reseedSSLProxyType=SOCKS5
router.reseedSSLRequired=false
time.sntpServerList=127.0.0.1
i2np.laptopMode=true
```

**Disable the Outproxy**

**We don’t want to access the Clearnet with I2P run the following to disable Outproxies**

**Remove the outproxy from the tunnel on port 4444**

`sudo sed -i '/^.*tunnel\.0\.\(proxyList\|option\.i2ptunnel\.httpclient\.SSLOutproxies\)/d' "/var/lib/i2p/i2p-config/i2ptunnel.config"`

**Disable the https outproxy (port 4445)**

`sudo sed -i 's|^.*\(tunnel\.6\.startOnLoad\).*|\1=false|' "/var/lib/i2p/i2p-config/i2ptunnel.config"`

**Changing I2P’s listening interface**

**I2P listens for connections on 127.0.0.1. This won’t work for us since we want to access I2P from the Workstation.
We’ll setup I2P to listen on the Gateway IP, which could be 10.137.x.10 depending on the Whonix version that you’re using. Note:
By the time we’re finished here, you will be able to access I2P from the workstation via 127.0.0.1 as well.**

`GATEWAYIP=$(ip addr | grep 'eth1' | grep -v 'BROADCAST' | cut -d / -f 1 | awk '{print $2}')`

`sudo sed -i "s/\(.*interface=\).*/\1$GATEWAYIP/g;s/\(.*targetHost=\).*/\1$GATEWAYIP/g" /var/lib/i2p/i2p-config/i2ptunnel.config`

`sudo sed -i "s/127\.0\.0\.1/$GATEWAYIP/g" /var/lib/i2p/i2p-config/clients.config`

`echo -e "susimail.host=$GATEWAYIP" | sudo tee /var/lib/i2p/i2p-config/susimail.config > /dev/null`

**change the Router console listening IP back to localhost**

`sudo sed -i "s/clientApp\.0\.args\=7657 \:\:1\,$GATEWAYIP/clientApp\.0\.args\=7657 \:\:1\,127\.0\.0\.1\./g" clients.config`

### Whonix Workstation (Template)VM

**Install Privoxy**

`sudo apt-get install privoxy`

**Edit the** `/etc/privoxy/config` **add i2p forwarding**

```
forward .i2p 127.0.0.1:4444
accept-intercepted-requests 1
max-client-connections 512
trustfile trust
```
**if you want to use Zeronet add the Line below**
```
forward .bit 127.0.0.1:43110
```
**change the line below to enforce the trust mechanism and restart privoxy**
```
enforce-blocks 0 to 1
```
**Edit the newly created file and add the lines below:**

`kdesudo kwrite /etc/privoxy/trust`

```
~*.i2p
~*.*.i2p
~.torproject.org
```


**Forwarding Whonix-Workstations Ports to Whonix-Gateway local Ports**

**Open** `/etc/anon-ws-disable-stacked-tor.d/50_user.conf` **with a editor in your Worksation-Template and insert the following:**

```
I2P_PORTS="2827 3456 4444 4445 6668 7622 7650 7651 7654 7656 7658 7659 7660 7661 7662 8998 8118"

for i2p_port in $I2P_PORTS ; do
   $pre_command socat TCP-LISTEN:$i2p_port,fork,bind=127.0.0.1 TCP:$GATEWAY_IP:$i2p_port &
done
```

### Whonix Workstation (App)VM

**Edit TorBrowser to use Privoxy**

**Open the TorBrowser enter** `about:config` **and change these Settings**

```
extensions.torbutton.use_nontor_proxy;true
network.proxy.no_proxies_on;0
network.proxy.http;127.0.0.1
network.proxy.http_port;8118
network.proxy.socks;         <--(blank)

```

### Debug
check your wrapper & router log for any errors
```
sudo tail -f /var/log/i2p/wrapper.log
sudo tail -f /var/log/i2p/log-router-0.txt
(if log-router-0 is showing old info try log-router-1)

```
check i2p's config files
```
sudo nano /var/lib/i2p/i2p-config/router.config
sudo nano /var/lib/i2p/i2p-config/clients.config

```
