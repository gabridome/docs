# Setting up an Electrum server reachable only by you on Tor.

This small guide weill help you to set up your own Electrum Server and 
to make it available on Tor network so that only your client is able to reach it.
The necessary steps are:
1. Set up Tor on your server (it can be your desktop provided that it has the necessary requirements)
2. Install and have your bitcoin node synchronized
3. Install Electrs and let it sync with your Bircoin Core node.
4. Set up an Hidden Service to make the Elelectrum Server Service reachable from outside in an authenticated way
5. Configure your client to access the authenticated service

This guide will redirect you to the official documentation of the single packets for correctness when necessary.

## Requirements
I would suggest at least 350 Gb of free disk space to devolve to this service (300 Gb for bitcoind and 50 Gb for Electrs). 
2 Gb of free RAM should be enough.
Tor 0.3.2.2 or above allow the creation of Version 3 hidden services and addresses. This guide refers to this kind of services.

## 1. Set up Tor on your computer
to use tor you have to have tor installed an running.

These instructions are for debian derived distribution of Linux. Please refer to the [Official Documentation](https://www.torproject.org/docs/documentation.html.en)
for other platforms.

`sudo apt install tor`

then `/etc/init.d/tor start` or `sudo systemctl start tor` Depending on your system configuration.
If new to tor you might not change the default setting.
To keep The safe default with minimal harassment (See Tor FAQ) just check that this line is present in the file:
ExitPolicy reject *:* # no exits allowed

this does not affect your services, listen, etc.. It will only prevent that you become a full exitpoint. 
Only enable this if you are sure about the implications.

## 2. Install and have your bitcoin node synchronized
To download and Install your bitcoin node, please follow the [Documentation](https://bitcoincore.org/en/download/).
To speed up the synchronization process it would help a lot to dedicate a lot of RAM to this phase.
A good starting point could be `bitcoind -daemon -dbcache=1000000`. This value depends on how much free RAM you can count on.
Once synchronized you can omit the dbcache parameter.
Depending on the resources available the synchronization could take hours or days so be patient.

## 3. Install Electrs and let it sync with your Bircoin Core node.
The [repository to clone Elctrs](https://github.com/romanz/electrs) has also the instructions to 
[install it](https://github.com/romanz/electrs/blob/master/doc/usage.md).

In few hours your Electrum server should be synchronized and usable on localhost. 
It could be a wise idea to test the server on localhost if you have installed the server on the same machine you 
have the Electrum client installed. In which case you can run Electrum and simply connect to your server by going on
Tools/Network/Servers, deselecting "Select servers automatically" and setting server to "127.0.0.1" and port "50001".
If the led on the botton right turns green, you are connected on your local electrum server.

Now we want to expose it on the Tor Network for our eyes only.

## 4. Set up an Hidden Service to make the Elelectrum Server Service reachable from outside in an authenticated way

To have an authenticated .onion address your Electrum client can connect to
it is necessary to add these lines in the /etc/tor/torrc file (or wherevever this file is located on your platform):

HiddenServiceDir /var/lib/tor/electrs/
HiddenServiceVersion 3
HiddenServicePort 50001 127.0.0.1:50001
HiddenServiceAuthorizeClient stealth homeDesktop,OfficeDesktop,laptop 

The way Tor hidden services work is that they create a tunnel across routers and firewall and open a global 
port for the service which is mapped to an internal port on 127.0.0.1 (or on the IP you have your Tor bind).
In particular, the line 

`HiddenServiceAuthorizeClient stealth desktopDiCasa,DesktopUfficio,laptop`

will create 3 .onion addresses and three passwords you will be able to use with each of 
your **clients** to allow it to connect it in an authenticated way (we will see below how to set up 
these clients to use the passwords). 

In this case above, I have decided to create, 3 accesses for three different clients that I have denominated homeDesktop, OfficeDesktop and laptop.

The hidden Electrum server service will be reachable at port 50001 (global port) of the .onion address, which will be created at the restart of the Tor service. Both types of addresses can coexist on the same node.

Save the file and restart the Tor service. In linux:
`/etc/init.d/tor restart` or `sudo systemctl restart tor` depending on the configuration of your system.

You will find the newly created address with:

sudo cat /var/lib/tor/electrs/hostname 
in the case of a version 3 Tor address.

You will find something like:

```
vonwnx37qz7pczophtlw7khrwr7enh6iv3fotgvj23j5jk3zikbcktyd.onion
fagcayfgaibflgbgyggcgYGbBygUBbuFukfbFUfVuyvuVvVUYGUVgyuy.onion pcDsdhaeiOwUlvokeneWlJ # client: homeDesktop
UYDRiRIroXrotrtorxtrxorrhfyjedwrxrkrdheukfdfytjyedTutydd.onion wobfljkaeiwwfpdnvosbda # client: OfficeDesktop
fkfhfDFLgkvtfULlkfgluyjfkdjdkhdrtjfKfjJDjgfscdhshgjHDSHd.onion glwcbllulzSAEhhcHgcwxw # client: laptop
```

As we wrote, in this case we have created three accesses for three clients.
In each line is reported for each client:

* the address to which your Electrum client must connect to on the port 50001
* the password you have to insert in the tor configuration file `/etc/tor/torrc`.

So now we are ready to access the service from our client.

## 5. Configure your client to access the authenticated service

Once we have the Electrum client installed on our office desktop, we refere to the corresponding line
of the output above 

`fagcayfgaibflgbgyggcgYGbBygUBbuFukfbFUfVuyvuVvVUYGUVgyuy.onion wobfljkaeiwwfpdnvosbda # client: OfficeDesktop`

We open the config file of our Electrum installation (normally ~/.electrum/config),

We also set (if present) or create the following keys to the reported values:

```
"auto_connect": false,
"oneserver": true,
"proxy": "socks5:127.0.0.1:9050::",
"server": "fagcayfgaibflgbgyggcgYGbBygUBbuFukfbFUfVuyvuVvVUYGUVgyuy.onion:50001:t",
```

Once saved we are ready to insert the password for our service in the Tor configuration file:

`HidServAuth fagcayfgaibflgbgyggcgYGbBygUBbuFukfbFUfVuyvuVvVUYGUVgyuy.onion wobfljkaeiwwfpdnvosbda`

In this way, once we connect to inejmxtkeokhbue.onion via Tor, our device knows that it has to
use the password wobfljkaeiwwfpdnvosbda to access the service.

We save the file and we restart the Tor service on the client.

From now on, when we start Electrum it connects ONLY with our server at inejmxtkeokhbue.onion via Tor
and use the right password to access.

We can configure the other client in the same way.
