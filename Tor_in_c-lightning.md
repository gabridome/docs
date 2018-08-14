# HOWTO USE TOR WITH C-LIGHTNING

to use Tor you have to have Tor installed an running.

i.e.
```
sudo apt install tor
```
then `/etc/init.d/tor start` or `sudo systemctl start tor` Depending 
on your system configuration.

If new to tor you might not change the default setting.

To keep The safe default with minimal harassment (See [Tor FAQ])
just check that this line is present in the file:

`ExitPolicy reject *:* # no exits allowed`

this does not affect c-lightning connect, listen, etc..
It will only prevent that you become a full exitpoint.
Only enable this if you are sure about the implications.

If we don't want to create .onion addresses this should be enough.

There are several way by which a c-lightning node can accept or make connections over Tor.

## Using Tor as a socks5 proxy for outbound connections

With Tor installed and running, a node can connect to the .onion 
address of a Tor enabled node if the `lightning`daemon has been started with the 
option `--proxy=127.0.0.1:9050`. 
If `--always-use-proxy` is added, the proxy will be used also to connect 
to a IPV4 o IPV6 address.

## How to be reached

### IPV4 and IPV6 addresses

To expose the node to Clearnet connections, the node has to be bound to an 
an address and a port. To configure the node correctly it is necessary to 
understand if the node is inside an internal network.

In linux:

Discover your external IP address with: `curl ipinfo.io/ip`

and your internal IP Address with: `ip route get 1 | awk '{print $NF;exit}'`

If the two address differ, the node is inside an internal network.

The node must be started with the option

`--bind-addr=internalIPAddress:port`

If the node has to be announced on the outside network

`--announce-addr=externalIPAddress:port` has to be added.

If the two address (internal and external) are the same, the node is exposed 
to the network and it is not on an internal network.

The option `--bind-addr=externalIPAddress:port` can be used to avoid 
announcing the IP address to the network.

To bind and announce the address the option `--addr=externalIPAddress:port`
is used.

To connect to these nodes from an other node:

`lightning-cli connect nodeID externalIPAddress port`

### On Tor

To be present on the Tor network it is necessary to have the service on one or  
more .onion address.

#### Nodes on a persistent Tor .onion address

A persistent .onion address is a node that doesn't change across service 
restarts.

To have a persistent .onion address other nodes can connect to, it 
is necessary to set up a [Tor Hidden Service].

To do that we will add these lines in the `/etc/tor/torrc`file:

````
HiddenServiceDir /var/lib/tor/lightningd-service_v2/
HiddenServicePort 1234 127.0.0.1:9735
````

If we want to create a version 3 address, we will add also `HiddenServiceVersion 3` so
the whole section will be:

````
HiddenServiceDir /var/lib/tor/lightningd-service_v3/
HiddenServiceVersion 3
HiddenServicePort 1234 127.0.0.1:9735
````

The hidden lightning service  will be reachable at port 1234 (global port)
of the .onion address, which will be created at the restart of the 
Tor service. Both types of addresses can coexist on the same node.

Save the file and restart the Tor service. In linux:

`/etc/init.d/tor restart` or `sudo systemctl start tor` depending 
on the configuration of your system.

You will find the newly created address with:

`sudo cat /var/lib/tor/var/lib/tor/lightningd-service_v2/hostname` or

`sudo cat /var/lib/tor/var/lib/tor/lightningd-service_v3/hostname` in the 
case of a version 3 Tor address.

To bind the `lightningd`service to the hidden service if the 
hidden servise string is `HiddenServicePort 1234 127.0.0.1:9735` add the 

`--bind-addr=127.0.0.1:9735` option to the `lighningd` service. 
The service will be reached at the .onionAddress:1234 with the command
`lightning-cli connect .onionAddress 1234`

#### Nodes on a non-persistent Tor .onion address

To provide the node a non-persistent .onion address it
is necessary to access the Tor auto service. These types of addresses change 
each time the Tor service is restarted.

To create and use the auto service follow this steps:

Edit the Tor config file `/etc/tor/torrc`

You can configure the service authenticated by cookie or by password:

##### Service authenticated by cookie 
We add the following lines in the `/etc/tor/torrc` file:

````
ControlPort 9051
CookieAuthentication 1
CookieAuthFileGroupReadable 1
````

##### Service authenticated by password 

In alternative to the CookieFile authentication. you can set the authentication 
to the service with a password by following theses steps:

1. Create an hash of your password with `tor --hash-password yourpassword`.
This returns a line like

`16:533E3963988E038560A8C4EE6BBEE8DB106B38F9C8A7F81FE38D2A3B1F`

2. put these lines in the `/etc/tor/torrc` file:
```
ControlPort 9051
HashedControlPassword 16:533E3963988E038560A8C4EE6BBEE8DB106B38F9C8A7F81FE38D2A3B1F
```
Save the file.

To activate these changes:

`/etc/init.d/tor restart`

The auto service will be used by adding `--bind-addr=autotor:127.0.0.1:9051` if we don't want to announce the service on the network.

To announce the non persisten address on the network, it is necessary to use 
the `--addr=autotor:127.0.0.1:9051` option instead.

In the case the auto service is authenticated through the password, it will 
be necessary to add the option `--tor-service-password=yourpassword` (not the hash).

The created non-persistent .onion address wil be shown by the `lightning-cli getinfo` command. 
The others nodes will be able to `connect` to this .onion address through the 
9735 port with the command `lightning-cli connect peerID .onionAddress 9735`


Some examples:
Internal IP address: internalIP
Internal port the service is listening to: internalPort
External Ip address: externalIP
External port the service is listening to: externalPort
Non-persistent .onion address: nonpersistent.onion
persistent .onion address: persistent.onion
Internal port the node is listening to: internalPort
global port of the hidden service
```
lightningd --bind-addr=autotor:127.0.0.1:9051 --proxy=127.0.0.1:9050 --always-use-proxy
```
Even if nobody know the address, the node can always `connect`to an other 
node and get the state of the network.
To know the .onion address (nonpersistent.onion): `lightning-cli getinfo`.
A node, which knows somehow nonpersistent.onion, will connect through 
the socks5 proxy to Tor and non Tor enabled nodes.
```
lightning-cli connect peerID nonpersistent.onion 9735
```
Note that the port is always 9735 in this case.

```
lightningd --addr=autotor:127.0.0.1:9051 --proxy=127.0.0.1:9050 --always-use-proxy
```
The node .onion address is announced on the network (the port 
is always 9735 in this case). Every outgoing communication is passing through 
the Tor socks5 proxy.
```
lightningd --bind-addr=internalIP:internalPort --proxy=127.0.0.1:9050 --always-use-proxy
```
It is not necessary to announce either external IP or .onion addresses: 
They can be redirected or by NAT (IP address), or by the line in which we 
create the hidden service:

````
HiddenService globalPort internalAddress:internalPort
````
