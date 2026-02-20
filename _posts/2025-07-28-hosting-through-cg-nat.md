---
layout: post
title:  "Self-hosting services behind Carrier Grade NAT"
subtitle: "How to save money on a static IP"
author: Andy
category: networking
date: 2025-07-28 01:00:00
cover: '/assets/images/2025/07/28/wireguard_traffic.png'
---

# A problem

A couple of months ago I decided that I'd finally get round to doing some self-hosted stuff. I had a few ideas, but it had been a long time since I played around with exposing any part of my home network to the internet. This was due to a few reasons. Firstly - I have very little time, with a busy work life and an even busier home life. Secondly, I subscribe to the doctrine that "it isn't paranoia if they really are out to get you". We should all be reasonably paranoid about opening up ports to the masses and putting our infrastructure at risk.

That said, I also have a healthy resentment of hosting *everything* elsewhere, and giving even more power (and money) to the tech bros. It's not that I'm jealous of their wealth (ok maybe a bit), but I do feel that the Metas, Googles, Apples, Microsofts and others can see far more of our personal data than any government agency and that makes me a little uncomfortable.

I digress... the point is that I wanted to start self-hosting a few common services for myself and my family to use. How hard could it be?

{% include image-link.html lightbox="under-construction-1" image_path="/assets/images/2025/07/28/under_construction.gif" alt_text="Under construction image"  caption="I'm going to bring these back" width="50%" %}

For most people, it really isn't hard. Let's use the example of a web service. If you want to host your own website, it could be as simple as putting a PC in your router's De-militarized Zone (DMZ), which will allow all traffic through the firewall just to that PC. Once that's done you could install a simple web server, upload some files to your web root directory and away you go. We'll ignore the fact that, without significant tweaks, this could be a bad decision from a security perspective for now.

The next thing you'd need to do is tell people how to get to your shiny new site. This would normally be via a domain name like `www.myawesomedomain.xyz`. Domain names are just human readable signposts to an IP address such as 20.26.156.215 (github.com). So it stands to reason that you'd need to know the IP address of your router's internet or Wide Area Network (WAN) connection to be able to link a human readable name to that address.

# Some History

For home internet connections, you are normally assigned a public IP address from your Internet Service Provider (ISP). Most people neither know nor care what that IP is, but the architects of the internet intended that address to be like a house address. If someone visits it, they arrive at your "front door". As the internet grew from its humble beginnings, it was discovered that the number of unique IP addresses available were finite, due to the way in which they are constructed. Furthermore, the Internet Assigned Numbers Authority (IANA), had dished them out in such vast numbers in the early days (the UK's Department of Work and Pensions owned 16 million), there was a genuine fear they were going to run out.

There's a load of history and detail I'm skipping for brevity, but one of the consequences of this is that your router is still generally assigned a random IP address for a period of hours or days or weeks, but it isn't *yours* and can change. This is a problem if your domain name `www.notarealsite.xyz` was pointing to your computer on 1.2.3.4 but then your router's public IP changes to 4.5.6.7. Disappointed visitors would see something similar to this:

{% include image-link.html lightbox="trouble-finding-1" image_path="/assets/images/2025/07/28/trouble_finding.png" alt_text="Firefox when you browse to an unknown address"  caption="It took me a few goes to find an unregistered name..." width="75%" %}

That's not ideal. So how can the aspiring self-hoster get round this? Well, one answer is requesting a *static IP address* from your ISP. This means you say "would you mind terribly assigning me a fixed IP address so that people can find me at this address from now until the end of time?". Assuming you are with a reasonably accommodating ISP they may do this for free, but usually there is a charge (capitalism, yay!). Here's a random selection of prices for this service from a few common ISPs today:


<div align="center">
<table>
<thead>
<tr class="table">
<th>ISP</th>
<th>Static IP available</th>
<th>Price</th>
</tr>
</thead>
<tbody class="table">
<tr>
<td>BT</td>
<td>No (Unless business)</td>
<td>-</td>
</tr>
<tr>
<td>Virgin Media</td>
<td>No (Unless business)</td>
<td>-</td>
</tr>
<tr>
<td>Sky</td>
<td>No (Unless business)</td>
<td>-</td>
</tr>
<tr>
<td>Gigaclear</td>
<td>Yes</td>
<td>£2 per month</td>
</tr>
<tr>
<td>Airband</td>
<td>Yes</td>
<td>£12 per month</td>
</tr>
</tbody>
</table>
</div>


I have to admit, this picture is quite different from the last time I looked at static IP availability, but it makes sense when you get into it. Static IPs, for the reasons outlined above, should be expensive, like any scarce resource. Also... if you happen to be with Airband because <s>they are the only game in your particular town</s> they are the best, you're faced with paying £12 extra per month for the privilege. That's 3 Happy Meals dammit! That said, the reason they aren't generally even offered to residential customers is mostly due to lack of demand, which itself has decreased even from us nerds, due to a nifty little alternative to static IP addressing called Dynamic DNS (DDNS).

{% include image-link.html lightbox="3-happy-meals-1" image_path="/assets/images/2025/07/28/3_happy_meals.jpg" alt_text="Three Happy Meals"  caption="The monthly cost of a static IP with Airband" width="75%" %}

# Dynamic DNS

Dynamic DNS (DDNS) is a reasonably simple (but effective) solution to maintaining a public internet presence when the IP address of your home router keeps changing. Companies that offer DNS services allow users to register a domain name and provide a way for your router to regularly (say every few minutes) send its *current* public IP address to them. When they receive this information, the company updates internal routing information to say "`www.myawesomedomain.no-ip.org` was at 1.2.3.4, but is now at 5.6.7.8". With this method of notifying the DNS record holder of any changes, it's possible for you to have a consistent name to IP address mapping, to allow anyone to visit your self-hosted site. In basic terms it works like this:

- Visitors to your website enter `https://www.myawesomedomain.no-ip.org` in their browser
- The browser uses the traditional domain name resolution system (DNS) to ask an upstream source (often your ISP or Google or Cloudflare) to provide an IP address.
- The upstream source works out that they don't hold the answers for the domain no-ip.org, but they can forward the request to that company
- no-ip.org receives the request and looks up myawesomedomain's current IP address and returns this back up the request chain all the way to your browser

There are a number of free DDNS providers with various options and fees if you need more than the basic offerings. Here are some common ones.


<div align="center">
<table>
<thead>
<tr class="table">
<th>Provider</th>
<th>Cost</th>
<th>Domain name</th>
</tr>
</thead>
<tbody class="table">
<tr>
<td>No IP</td>
<td>Free (for one domain)</td>
<td span="markdown">&#42;.no-ip.org</td>
</tr>
<tr>
<td>Free DNS</td>
<td>Free (for 5 subdomains)</td>
<td span="markdown">&#42;.afraig.org</td>
</tr>
<tr>
<td>Duck DNS</td>
<td>Free</td>
<td span="markdown">&#42;.duckdns.org</td>
</tr>
</tbody>
</table>
</div>


This works wonderfully for many, many people... but not for me 🤦🏻‍♂️  That's because my ISP uses something called Carrier Grade Network Address Translation or CGNAT for short.



# CGNAT

Carrier Grade NAT was designed to help with the problem of a limited IPv4 address space (that shortage of addresses). If you take a brief look at the [Wikipedia article](https://en.wikipedia.org/wiki/Carrier-grade_NAT), you'll likely note that this is the *only* listed advantage, followed by a long list of disadvantages. To be fair, the "disadvantage" of your router not having a unique public IP address reachable from the internet, could also be viewed as an advantage. Nefarious types can no longer port scan your router for open services or indeed reach your router at all from the public internet unless in response to an outbound connection. That said there isn't much else to like about this.

The main issues is that your ISP has a connection to the internet (and a public IP address) which it shares between all of its customer's routers. Each customer router (Consumer Premises Equipment or CPE) is allocated a private non-routable IP address and the ISP keeps track of outbound requests from the customer's router like this:

1. The customer's router sends a request for a service on the internet (e.g. `www.google.com`) from its WAN IP of 10.1.2.3.
2. Once the name of the service has been converted to an IP address, the ISP's external gateway to the internet rewrites the *source* address of this request to its public (routable) IP address 200.4.5.6.
3. Google sends its response to 200.4.5.6 and the ISP external gateway checks its internal routing table to determine who requested this. It finds it was the customer's router so re-writes the *destination* address of the response to 10.1.2.3 and sends it to the customer.

This is the Network Address Translation (NAT) part and it is transparent to regular users. Users just type `www.google.com` in their browser and magically get presented with Google's search page. Cool.

Although, when you *want* to have a public presence on the internet it isn't cool. Knowing that a static IP address costs extra money, one might think about using DDNS, except you *can't* do that with CGNAT. DDNS works by your router (or a computer behind it) frequently updating a DNS provider with your router's **public** IP address... but now it never gets one. If you were to run a DDNS client, you'd be sending a private and non-routable IP address to the DNS provider. Damn. We're going to need an alternative solution.

# Commercial VPNs

It is worth saying that there are a few polished and 'free' solutions to this problem in the form of commercial Virtual Private Networks (VPNs). These are designed to allow you to securely connect devices and networks over the internet and in doing so could allow you to bypass CGNAT by creating a tunnel from an endpoint on the web to a machine in your network.

Two of the best known providers are [ZeroTier](https://www.zerotier.com/) and [Tailscale](https://tailscale.com/). Both of these companies have similar offerings which include multiple price plans (including free versions) along with support for multiple platforms (e.g. Mac, Windows, Linux, iOS, Android, etc.). In both cases you sign up via email or common identity providers (e.g Microsoft, Google, Apple, etc.), download the relevant application for any clients you want to connect and away you go.

I wasn't too keen on setting up yet more accounts (or logging in with Google credentials) and I resented the fact that the 'free' tier was limited to a handful of devices. Instead, I noted that Tailscale used a VPN technology called WireGuard under the hood, so I decided to cut out the middle person and use this directly.

# WireGuard

Traditionally, I might have used something like [OpenVPN](https://openvpn.net/) to create a secure site-to-site connection from a node in my home network to a point of presence out on the internet (such as a VPS) in order to provide a secure tunnel into my network.

That said, I was interested in trying something different and I'd heard good things about [WireGuard](https://www.wireguard.com/), which claims to use state-of-the-art cryptography ("reviewed by cryptographers"!), to provide a fast and modern VPN that has a much simpler implementation than OpenVPN/OpenSSL. It has also been incorporated into the Linux kernel for the past 5 years which is a good indicator of stability.

{% include image-link.html lightbox="wireguard-1" image_path="/assets/images/2025/07/28/wireguard.svg" alt_text="WireGuard logo"  caption="Also the logo has a dragon!" width="75%" %}


WireGuard securely encapsulates IP packets over User Datagram Protocol (UDP) to transport its data across the internet. In practice this means that it encapsulates both application data but also the header information containing instructions on where the packet is destined for (and came from). This allows it to be used as a Virtual Private Network (VPN) which means you can securely extend private networks (like internal home or business networks) to anywhere in the world.

This encapsulation method isn't particularly novel, but the way in which it secures the traffic is. WireGuard uses a technique called *Cryptokey Routing*, whereby each network interface on the server has a private (secret) key and a list of peers (remote computers) that is can connect to it, along with each peer's public key.

Let's consider two machines on opposite sides of the world that wish to communicate securely using WireGuard.

A VPN endpoint has an interface (wg0) which has a public-key, a private-key and listens on a particular UDP port (41414). The interface also has a list of peers, each of which is identified by their public keys and each of which has a list of 'allowed' source IPs.

{% include image-link.html lightbox="wireguard-example-1" image_path="/assets/images/2025/07/28/wireguard_example.png" alt_text="WireGuard config"  caption="Source: https://www.wireguard.com/papers/wireguard.pdf" width="100%" %}

When this endpoint wants to send a packet to one of these peers, it looks up which Peer Public Key to use, by matching the destination address of the packet with the table above. So for 10.10.10.230, the public key beginning `gN65...` would be used to encrypt the packet.

Equally, the peer at 10.10.10.230 would hold the corresponding private key and would therefore be able to decrypt that packet. The same process works in reverse - if the peer at 10.10.10.230 was to send a packet in response, it would have a similar table, where the IP address of our endpoint would be listed with the Public Key `HIgo...`.

There are many benefits to this approach, including built-in peer authentication, simplified firewall rules and perfect forward secrecy. In addition it is incredibly simple to set up.

Before we go through the installation process we need a machine on the internet that will act as a 'VPN server' which we can connect to from our home network to establish an encrypted tunnel. This is where a Virtual Private Server (VPS) comes in.

## VPS

A VPS is essentially a bit of private space on a reasonably powerful server somewhere on the internet. You can be sharing a physical server with potentially hundreds of other users, but you only ever see your own stuff and it looks like you have a machine all to yourself (neat).

The price of a VPS can be as little as £1 per month. You can pay more money to get more space, additional processing power, etc., but as I'm looking to host everything on a PC at home and just use the VPS as a relay (forwarding requests and their responses), I don't need more than this.

{% include image-link.html lightbox="cheap-vps-1" image_path="/assets/images/2025/07/28/cheap_vps.png" alt_text="Cheap VPS related image"  caption="You can spend very little on a VPS and domain name" width="50%" %}

Renting a VPS gives me a server on the internet with a fixed IP address which I was also able to associate with a domain name I pay around £10 a year to rent. This post is a little long already so I won't go into the details of that, but there are plenty of guides by the VPS/DNS companies themselves. Essentially, I can now type (for example) `http://www.myawesomedomain.xyz` into my browser's address bar, it will go and ask a name resolution service for the associated IP address, which will return that of my VPS. The next step is to use WireGuard, to create an encrypted tunnel from a PC in my home to the VPS, the final link in the chain.

{% include image-link.html lightbox="mywebsite-1" image_path="/assets/images/2025/07/28/mywebsite.png" alt_text="Screenshot of a very basic website"  caption="Needs work..." width="75%" %}


## Installation

So we have a PC in our home network and we have a VPS with a public IP address out there on the internet. That's our two endpoints - let's go through the setup of WireGuard on the VPS side first.


{% highlight bash %}

$ sudo apt update
$ sudo apt install wireguard

{% endhighlight %}

Next you need to generate a public/private keypair for this peer (we'll call this the 'server'):


{% highlight bash %}

$ wg genkey | sudo tee /etc/wireguard/private.key
SSBsaWtlIHlvdXIgdGhpbmtpbmcsIGJ1dCBuby4uLgo=
$ sudo chmod go= /etc/wireguard/private.key

{% endhighlight %}

The second command removes access permissions to the private key for users and groups other than the root user. Next create the corresponding public key:


{% highlight bash %}

$ sudo cat /etc/wireguard/private.key | wg pubkey | sudo tee /etc/wireguard/public.key.
OhIZ0Hiw2A9Xg5SHhriN4XV8/PwqV0sF+bOBKH61I3Y=

{% endhighlight %}

Now that you have your public/private key pair you need to create your WireGuard configuration file as `/etc/wireguard/wg0.conf`. It needs to contain details of the 'server' interface, but also peers which are permitted to connect:


```
[Interface]
PrivateKey = SSBsaWtlIHlvdXIgdGhpbmtpbmcsIGJ1dCBuby4uLgo=
ListenPort = 51920
Address = 10.0.0.1/32

[Peer]
PublicKey = QSBtYWRlIHVwIHB1YmxpYyBrZXkuIHNlcmlvdXNseQo=
AllowedIPs = 10.0.0.2/32
```

So our server is listening for peers to connect on UDP/51920. We also have one peer listed which is identified with the public key starting `QSBt...` and it *has* to have the IP 10.0.0.2 (this is a point-to-point connection).

The configuration for a client peer (our PC at home) is very similar. Follow the steps above to install WireGuard, generate a public private key pair and then create a configuration file. The configuration file on the client peer is almost identical, but also needs to contain the IP address of the 'server peer' (VPS) under the 'Endpoint' entry:


{% highlight bash %}

[Interface]
PrivateKey = SSByZWFsbHkgYW0gbWFraW5nIHRoZXNlIHVwLiBob25lc3QK
Address = 10.0.0.2/32

[Peer]
PublicKey = OhIZ0Hiw2A9Xg5SHhriN4XV8/PwqV0sF+bOBKH61I3Y= 
AllowedIPs = 0.0.0.0/0
Endpoint = 123.456.789.10:51920
PersistentKeepAlive = 25

{% endhighlight %}

Finally, we're ready to fire up the VPN. On both machines we need to enable WireGuard as a service so it starts at boot time and also actually start the service for the first time:

{% highlight bash %}
sudo systemctl enable wg-quick@wg0.service
sudo systemctl start wg-quick@wg0.service
{% endhighlight %}

A quick check using `wg show` will give you the status of the server peer:

{% highlight bash %}

$ sudo wg show
interface: wg0
  public key: OhIZ0Hiw2A9Xg5SHhriN4XV8/PwqV0sF+bOBKH61I3Y=
  private key: (hidden)
  listening port: 51920

peer: QSBtYWRlIHVwIHB1YmxpYyBrZXkuIHNlcmlvdXNseQo=
  endpoint: 111.112.113.114:36007
  allowed ips: 10.0.0.2/32
  latest handshake: 41 seconds ago
  transfer: 20.65 MiB received, 14.11 MiB sent

{% endhighlight %}

Running the same command on the client peer should give you:

{% highlight bash %}
interface: wg0
  public key: QSBtYWRlIHVwIHB1YmxpYyBrZXkuIHNlcmlvdXNseQo=
  private key: (hidden)
  listening port: 36007
  fwmark: 0xca6c

peer: OhIZ0Hiw2A9Xg5SHhriN4XV8/PwqV0sF+bOBKH61I3Y=
  endpoint: 194.164.91.231:51920
  allowed ips: 0.0.0.0/0
  latest handshake: 2 minutes, 18 seconds ago
  transfer: 14.11 MiB received, 20.66 MiB sent
  persistent keepalive: every 25 seconds
{% endhighlight %}

Awesome! From our VPS we can now send packets to a PC in our home network via an encrypted tunnel. This has partially achieved the goal of bypassing CGNAT, but we still need to tweak the configuration to allow only specific services (e.g. web traffic) down the tunnel and setup some port forwarding so that when a user connects to a port (e.g. TCP/80) on our VPS, the traffic is automatically sent through the encrypted tunnel to the same port on a PC inside our home network.

Next time, I'll detail how to do that along with how I used this setup to replace Google Calendar with an open-source equivalent, so my family could share their calendars without having to also share with Google.

