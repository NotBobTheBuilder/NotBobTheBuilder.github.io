# For once, not DNS: Huawei Modem ARP spoofing

A couple of days ago I noticed my Ring Chime was disconnected in the Ring app. After a couple of years happily connected it was time to fulfil the ultimate purpose of both software and hardware: annoy humans.

![image](https://user-images.githubusercontent.com/1965587/106320736-2e22fe00-626b-11eb-8e03-8c103614f1ee.png)

I went through the reconnection process a couple of times to no luck. I called Ring support, who took me through the same process a couple of times to the same result. But one step in their process yielded an unusual result: if I set up a WiFi hotspot on my phone, the Chime would connect to it!

But when renaming my home WiFi network to have the same name and password as the hospot, the Chime still can't connect.

All the while my Doorbell has no problems on my WiFi.

After running through all their troubleshooting, Ring support were convinced it had to be related to my router or ISP. This was _probably_ a reasonable conclusion on their part given the hotspot trick worked, but left me unsatisfied: my router runs [OpenWRT](https://openwrt.org/), so I know exactly what it's doing, and my ISP ([Zen](https://www.zen.co.uk/)) have never struck me as the type for shenanigans. Especially not shenanigans which only affect one particular device within the LAN.

### Stuck.

This is the point I found myself stuck for a couple of days. The Chime reports no diagnostics logs (at least, none that are visible to the customer). More than a little frustrating, and ultimately one of the reasons I've only been buying new IOT/smart home devices that are compatible with [Tasmota](https://tasmota.github.io/docs/) and [Homebridge](https://homebridge.io/) (which along with comparable projects are excellent by the way).

All this aside I could see from my router that the device was connecting to wifi, and was getting an IP address. It seems like the connection was failing when it tried to phone home.

### It's got to be DNS.

I mean come on right? The problem has all the properties you expect to be DNS' fault. Spooky change that only affects one device? Sounds like DNS. Got an IP and internet connection but can't phone home? Sounds like DNS. Works on some networks and not others? Surely DNS?

To keep pursuing this theory, I already had a couple of router configuration settings that help me out - DHCPD on my router will always give a fixed address to my Chime (10.0.1.2), and I run BIND on the router too to manage some internal zones so can see every query.

I turned on the `querylog` option within BIND and looked through the logs for queries from `10.0.1.2`, expecting maybe a `SERVFAIL` or something.

But there were no queries, at all. The device connects to wifi, asks for and gets an IP (verified in the DHCPD logs), then disconnects without doing any DNS lookups.

### What _should_ it be doing?

At this point I am very confused. For comparison I stand up a wireless hotspot on my macbook and use [Wireshark](https://www.wireshark.org/) to see what's going on, and to be 100% sure I get a connection, link the macbook hotspot to my iPhone's 4G connection.

As expected it phones home fine when not in my own network, and I now have at least some network logs to investigate.

The logs look as you might expect: get an IP, get the time, look up the ring servers in DNS, and connect.

![image](https://user-images.githubusercontent.com/1965587/106329127-d8555280-6278-11eb-9838-58e52ee8c0b2.png)

Why can't it do this on my own network?

At this point I am more than a little confused. I tried a few things in guesswork, like changing the network DNS servers to cloudflare (1.1.1.1) or google (8.8.8.8), or connecting with a static IP address. No luck

### Bring out the big guns

At this point my only remaining thought was to try wireshark on my own laptop when I connect to WiFi and see what happens.

This reveals something _weird_. Two ARP responses, with different MAC addresses, claiming to be the same IP for my router.

![image](https://user-images.githubusercontent.com/1965587/106311707-7affd800-625d-11eb-9345-15550da65e0b.png)

This is immediately promising. ARP sits below IP on the networking stack and allows devices to correlate IP addresses with MAC addresses of hardware in the same local network. If you want to send a message to an IP for the first time, you can use ARP to find that machines hardware address in the local network or find the hardware address of a router that will forward on your message.

Sending ARP responses which claim an IP address which isn't yours is [ARP Spoofing](https://en.wikipedia.org/wiki/ARP_spoofing). It's at best a pain in the ass, and at worst a security risk, because devices will start sending packets to the wrong place, and that wrong place may be malicious.

Looking at traffic from our imposter's MAC address we see even more bad form: this device is claiming every IP anyone tries to find.

![image](https://user-images.githubusercontent.com/1965587/106311711-7b986e80-625d-11eb-96cf-691c005730b4.png)

At this point my working theory is greatly narrowed down. The chime is connecting to wifi, getting an IP address, and then wants to connect to the internet, so uses ARP to find the router.

Our imposter then responds in the affirmative, claiming to be the router, but really doing nothing.

Our chime then sends its first requests to what it thinks to be the router, and after they go unacknowledged, gives up and disconnects.

Most network devices stick around long enough to build up an ARP cache whose entries get invalidated when new ARP messages come in. Over time, this means that though the imposter is claiming IP addresses, the real owner of an IP address sends its own messages, which overwrite the imposter.

In this case, this is probably a shortcoming of the Chime - it does not stick around long enough to see the real owner make itself known. But the chime is just a small corner of the problem now.

### Who is this imposter?

The first portion of a MAC address is registered to a particular manufacturer. In this case... no dice. The mac address is `2a:8a:1c:ea:51:c5`. It is now immediately obvious what the problem is!

(Just kidding)

In the address above, the locally administered bit is set, meaning the address is probably randomly generated. (You can spot a MAC address like this because the 2nd digit in hex form is 2, 6, a, or e) https://en.wikipedia.org/wiki/MAC_address#Universal_vs._local_(U/L_bit). Unlike other MAC addresses, entries in this spot can be locally generated, unlike other MAC addresses where the first portion is registered to a specific manufacturer.

Luckily, between my devices sits a managed switch, my handy [T1600G-28PS](https://www.tp-link.com/uk/business-networking/smart-switch/t1600g-28ps/) (more on that in a bit). Being a managed switch, one feature notes all the MAC addresses coming and going, and associates them in a handy table with which port they are linked into.

![image](https://user-images.githubusercontent.com/1965587/106322661-113bfa00-626e-11eb-8d66-fd692ebd14b4.png)

Tracking back the device connected to port 13, I find my modem, a Huawei MT992 [g.fast](https://en.wikipedia.org/wiki/G.fast) modem.

![image](https://user-images.githubusercontent.com/1965587/106322702-26188d80-626e-11eb-840f-c6d686ed7eb3.png)

Usually a modem wouldn't be connected directly to a switch as only a router needs to speak to it. In my case the switch is UPS backed, and powers the router through a [Power Over Ethernet](https://en.wikipedia.org/wiki/Power_over_Ethernet) [splitter](https://www.amazon.co.uk/TP-Link-Gigabit-Ethernet-Splitter-TL-PoE10R/dp/B003CFATQK), so if the power goes out my internet connection & WiFi (also driven by PoE access points) are unaffected.

I had originally isolated the modem into separate [VLAN](https://en.wikipedia.org/wiki/Virtual_LAN) based on it's MAC address. This keeps the router-modem traffic separate from the router-LAN traffic. But as you probably guessed - the spoofed MAC address changes from time to time, and so it snuck out of the VLAN originally envisioned for it, allowing it to spill its ARP-spoofs all over my LAN.

### Concluding

The fix I have settled with for now is using port-based VLANing. Now regardless of what MAC it uses the modem can only ever talk to the router. The modem won't be forwarded any ARP requests, and its own announcements will only reach the router.

Longer term the solution should be to connect the router directly to the modem with a PoE injector along the line, but with the VLANs now tightened up this is less urgent.

This was quite the detour given the original problem only manifested on one device. In reality there would have been some connectivity issues for devices connecting within the lan for the first time as all ARP requests would be pointed to the wrong place. However relatively quickly - within milliseconds - the false entries would have been getting invalidated by the real owners, so devices that stayed connected to the network would be routing to the right places.

It's also quite surprising the modem does this at all given it is an L2 device which _doesn't even speak IP_, and its MAC spoofing behaviour is even stranger. There is actually a sticker on the back of the device with a MAC address that it clearly doesn't use. I have no idea what this MAC address refers to as it is not used on the LAN.

Overall, this served as a big reminder that it is a pain in the ass to debug closed hardware. If you are interested in the IOT space, I really recommend looking at Tasmota, Homebridge, and their equivalents in the DIY smart home space. And likewise, tools like `tcpdump`, Wireshark, and even the basics like `arp` are worth learning because you never know when they may be useful.
