---
layout: post
title:  "Swapping Google Calendar for a self-hosted equivalent"
subtitle: "The magic of CalDAV"
author: Andy
category: networking
date: 2025-08-11 01:00:00
cover: "/assets/images/2025/08/11/calendar.jpg"
---

# Why would anyone want to do this?

In my last post, I set up the basic infrastructure to host web services from my home network. I had to perform a workaround due to the way my ISP provides my internet connection, but I am now in a position to host services and have a domain name to hand out to friends and family should I need to.

The first service I wanted to host was a shared calendar, so that my wife and I could manage our busy schedules (and the even busier schedules of our kids). Until recently, we had been using Google Calendar, which I must admit was terribly convenient. That said, I wrote previously about wanting to take some small steps into reclaiming ownership of some of my data. The majority of internet users are probably happy to use major service providers such as Google, Apple, Microsoft, etc. After all, these services have the benefit of "five nines" availability (i.e. 99.999% uptime), data resilience (a fire in one data center probably won't wipe out all your data) and are often completely freeeeeeee.

{% include image-link.html lightbox="google-calendar-1"  image_path="/assets/images/2025/08/11/google_calendar.webp" alt_text="Google calendar marketing image"  caption="Google Calendar: Excellent, if you're happy to share with Google" width="75%" %}


Except they aren't. The old adage "there's no such thing as a free lunch" still rings true today. If you dig into the T's & C's of these services you'll find that the provider is often quite clear that the data you upload to their services isn't necessarily *yours* anymore... you effectively give the provider permission to use it in any number of ways. Some of these appear positive and proactive (notifying you of suspicious account activity if something occurs that doesn't follow your regular pattern of life). Others are less positive, such as influencing you to make certain decisions (where to shop, who to vote for, etc.). Here's a snippet from Google's Privacy Policy at the time of writing:

<blockquote>
"We collect information to provide better services to all our users – from figuring out basic stuff such as which language you speak, to more complex things like which ads you’ll find most useful, the people who matter most to you online or which YouTube videos you might like."
</blockquote>

The keyword here is "like", Google collects information on its users and have provided some examples of how they might use it, but they can use it in any number of other ways. Whilst I'm not particularly comfortable with Google knowing who matters to me most online I appreciate that many people are fine with this. Each to their own.

Equally, if you could be bothered you could encrypt your data before uploading it to the cloud (in a file storage service for example) which would prevent much of this. Apple even give you an (optional) setting called Advanced Data Protection (ADP) for iCloud which uses end-to-end encryption, meaning allegedly not even Apple can read your data. I say allegedly... the UK Government recently took the unusual step demanding access to all Apple data, so Apple had to withdraw ADP. Ironically, this means that it was almost certainly a genuinely good privacy feature for UK-based Apple users... That said, I still want to try and carve out some small areas of privacy and I think the shared calendar paradigm is a good one to start with. So let's get into it.


# CalDAV

CalDAV stands for Calendar Distributed Authoring and Versioning. It is effectively a protocol to allow users to manage and share calendar data between multiple devices, and is widely supported by calendar applications. It is also based on WebDAV which was developed in the 90s to help move the web from static, read-only resources to a more interactive affair. In addition it utilises a media type called iCalendar, which is just a text-based object that contains multiple calendar properties as shown below.



{% include image-link.html lightbox="icalendar-1" image_path="/assets/images/2025/08/11/iCalendar.png" alt_text="An iCalendar object"  caption="An example of an iCalendar object (Copyright: Wikipedia)" width="75%" %}

CalDAV stores and shares calendar events in this format and manages the access control elements of multiple user (i.e. shared) calendars. As it uses WebDAV to provide that access, this implies it runs on a web server. So that's naturally what we need first.

# Wireguard and a basic Web Server

In my last post I talked about my ISP employing CGNAT which means I needed to stand up a Virtual Private Server (VPS) to act as a relay for any services I want to offer such as a web server. A server in my home network creates a Virtual Private Network (VPN) between itself and the VPS, so all I need to do is configure the VPN to allow the Hyper Text Transfer Protocol (HTTP), which browsers use to communicate with web servers, to reach my server.

It is worth mentioning that a web server can have multiple services running at the same time. A simple analogy is that of an apartment block. The building has one main address, but when the postie arrives in the lobby he'll find a bank of mailboxes, each with a different number on it, representing different apartments in that building. Similarly, a web server has a URL (address) and multiple ports (mailboxes) which represent different services (rather than apartments!). In the case of web site traffic, the "http://" is translated into port number 80, or more accurately, the browser is instructed to communicate with the remote computer using Transmission Control Protocol (TCP) and to connect to port 80 (TCP/80).

We'll go through setting up a proper web server in a minute, but for now I can run a simple program which is packaged with the Python scripting language to fire up a web server which listens on TCP/80 and serves a simple file like so:

{% highlight bash %}

mkdir /tmp/demo
cd /tmp/demo
echo "<p>Hello, World!" > index.html
sudo python -m http.server 80

{% endhighlight %}

The instructions above create a new temporary (until the next time I reboot this computer) directory with nothing else in it. The subsequent lines, change the working directory to the temporary directory and writes some Hypertext Markup Language (HTML) into a file called `index.html`. The last line switches the current user to the all-powerful 'root' user, just for this command, and runs a basic web server on TCP/80.

This change of user is temporarily required to bind a program (in this case Python's simple web server) to TCP/80. I could have picked any number between 1024 and 65535 and, as long as nothing else was already bound (listening) on that port, it would just work without needing to switch user. Ports numbering between 1 and 1023 are known as Privileged Ports and a normal user isn't allowed to host services on them. This was designed to stop malicious users pretending to be real services (like web servers on TCP/80). Later we'll see how a proper web server avoids running as the 'root' user - needless to say (or is it?), don't run real web servers as the 'root' user...

As a final step I need to tell my VPS that it is now acting as a 'router'. That is, it needs to forward packets to another computer. I achieve this using the following command:

{% highlight bash %}

sudo sysctl -w net.ipv4.ip_forward=1

{% endhighlight %}

So now if I type in "http://", followed by the IP address of the web server on my home network, into my browser I see:

{% include image-link.html lightbox="hello-world-1" image_path="/assets/images/2025/08/11/hello_world.png" alt_text="A screenshot of a basic web page"  caption="Python's http.server module in action" width="75%" %}


Now that our server is hosting a website (even if it contains only one web page that says hello), we can configure and test some changes to WireGuard to be able to access this website from the internet. On the computer running the web server we need to add the following lines to `/etc/wireguard/wg0.conf` under the `[Interface]` section (see [my previous post]({% post_url 2025-07-28-hosting-through-cg-nat %}){:target="_blank"} for details on WireGuard).


{% include note.html content="I spent too long trying to figure out why I couldn't reach port 80 on my VPS even after triple checking my VPS's firewall rules. It turns out that my VPS provider had its own firewall in front of my VPS, which I needed to modify via my VPS configuration panel." %}


### VPS (VPN Server) changes

The WireGuard configuration file provides the `PostUp` directive to run commands when the VPN tunnel is created (brought up) and `PostDown` directives to run commands when the VPN tunnel is destroyed (brought down). In my particular configuration I've chosen to use the `iptables` firewall administration tool to perform the following actions by adding the following directives below the existing `Address` directive:


```
PostUp = iptables -t nat -A PREROUTING -p tcp -i ens6 --match multiport --dports 80,443 -j DNAT --to-destination 10.0.0.2
PostUp = iptables -t nat -A POSTROUTING -o ens6 -j SNAT --to-source 111.112.113.114

PostDown = iptables -t nat -D PREROUTING -p tcp -i ens6 --match multiport --dports 80,443 -j DNAT --to-destination 10.0.0.2
PostDown = iptables -t nat -D POSTROUTING -o ens6 -j SNAT --to-source 111.112.113.114
```

The rules above instruct the firewall on the VPS to transparently route web requests down the VPN tunnel to the web server on the other end and receive responses. To achieve this the firewall needs to re-write the destination address (for requests) and the source address (for responses) on the fly. This is necessary because the clients on the internet have no knowledge of the tunnel or the IP addresses used in the private network, so this has to be managed behind the scenes.

### Web Server (VPN Client) changes

On the web server in my home network the additions to the `wg0.conf` file are even more simple:

```
PostUp = iptables -t nat -A PREROUTING -p tcp --match multiport --dports 80,443 -j DNAT --to-destination 192.168.1.146; iptables -t nat -A POSTROUTING -p tcp --match multiport --dports 80,443 -j MASQUERADE
PostDown = iptables -t nat -D PREROUTING -p tcp --match multiport --dports 80,443 -j DNAT --to-destination 192.168.1.146; iptables -t nat -D POSTROUTING -p tcp --match multiport --dports 80,443 -j MASQUERADE
```

Again, this is just telling the firewall on the web server that it will need to do some trickery to copy HTTP request traffic from one interface (with address 10.0.0.2) to another (with address 192.168.1.146). Also it will need to do the reverse (to allow traffic to go back down the tunnel to the VPS).

In both the VPS and web server configuration files, the `PostDown` directives just reverse the changes that were made to the firewall by deleting the rules that were added by the `PostUp` directive. 

With these changes, we can get to our "Hello, World!" page hosted by the Python simple web server from the internet, by using the URL or public IP address. Let's shut that down and replace it with a proper web server.

# Setting up Apache

I've chosen to use the Apache HTTP Server on my home machine. This is very much a personal choice and there are other options available such as [NGINX](https://nginx.org){:target="_blank"}.

Assuming, we have closed our temporary Python-based 'web server' (to free up TCP/80), for Debian-based Linux distributions (such as Ubuntu and Raspbian) getting started is as simple as running the follow two commands:

{% highlight bash %}

sudo apt install apache2
sudo service apache2 start

{% endhighlight %}

That's it! Now we have a web-sever running on TCP port 80. As I mentioned earlier, we have to install the Apache HTTP server using the 'root' user as it requires extra permission to open a service on that port. The difference between this and the `python` tool I used earlier, is that Apache doesn't process incoming requests within the same process as the one which sets up the socket (listening port).

This is important because it means Apache can hand off the processing of potentially dangerous user-data to another process which has very few privileges assigned to it. If that process was exploited by a malicious user, the idea is that the damage would be limited. To check this we can run the `netstat` command on our web server machine to see what is listening on certain ports (edited for brevity):


{% highlight bash %}

sudo netstat -lntp
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 0.0.0.0:443             0.0.0.0:*               LISTEN      682/apache2         
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      682/apache2

{% endhighlight %}

So the program `apache2` is listening on TCP/80 and TCP/443 (I'll get onto this in a moment). However, when we list all the running processes with the name `apache2` we see this:

{% highlight base %}

ps -fC apache2
UID          PID    PPID  C STIME TTY          TIME CMD
root         682       1  0 Jul16 ?        00:03:16 /usr/sbin/apache2 -k start
www-data  180570     682  0 08:30 ?        00:00:02 /usr/sbin/apache2 -k start
www-data  180571     682  0 08:30 ?        00:00:02 /usr/sbin/apache2 -k start
...

{% endhighlight %}

We can see the main process has the User ID (UID) of 'root', but there are multiple (again edited for brevity) sub-processes that are running as the lower-privileged user 'www-data' which will process the requests. Note that the sub-processes have a Parent Process ID (PPID) of `682` which matches the Process ID (PID) of the main Apache process.

So if we move our `index.html` that we created earlier to Apache HTTP Server's default **web root** `/var/www/html` we *should* now be able to get the "Hello, World!" page to load using the domain name of our VPS (e.g. `www.myawesomedomain.xyz`) from the internet!

{% include image-link.html image_path="/assets/images/2025/08/11/hello_world_2.png" alt_text="A screenshot of a basic web page"  caption="We've gone public!" width="75%" %}

Whilst it is always nice to be a published author (!) that ominous "Not Secure" warning is a bit of a buzzkill, so let's fix that.

# Communications Security

When the World Wide Web first became a thing in the early 90s, most websites didn't worry about who might be watching the network traffic between an end user and the web-site. I mean, what would be the point? Web pages were there to *publish* information. If someone wanted to eavesdrop on the communication between an end-user and that web site, they would just see the same thing the end-user saw when they visited the site with their browser.

Of course, over time, web sites became more interactive, which meant end-users were sending information to them rather than just downloading static pages of text. It started with simple data, a person's name or email address perhaps, but then websites started to have login pages which required a username and a password to access content. Anyone, who happened to own a computer (router) in the path that traffic took between client (user's browser) and server (website) could happily read all that data, steal passwords, personal messages, bank details, etc.

{% include image-link.html image_path="/assets/images/2025/08/11/netscape-navigator.jpg" alt_text="Netscape Navigator splash screen"  caption="Ahhhh those were the days" width="50%" %}


By 1996, the venerable Netscape Navigator browser (a genuinely magical bit of software in my mind), had introduced a mode of communicating called Secure Sockets Layer (SSL). Whilst it took many years for this technology to become prevalent, today you need to go out of your way to find a website that allows you to connect in an insecure fashion, where your data is sent 'as is' and not encrypted for transit. In fact, there are only a handful of websites that purposefully buck this trend such as [HTTP Forever](http://httpforever.com/){:target="_blank"} and the home of [the very first website](http://info.cern.ch/hypertext/WWW/TheProject.html){:target="_blank"} (Note: Ironically, if you click this link your browser will likely auto-redirect you to the HTTPS (secure) version, but you can manually remove the 's' and reload).

Anyway, the point is, any self-respecting website will have what is known as a Secure Sockets Layer (SSL), or more recently Transport Layer Security (TLS) certificate. This allows browsers and websites to encrypt their communications in such as way that only they can decrypt data that flows between them. The computers (routers) sitting on the communication path, forwarding this data, just see encrypted traffic.

## SSL/TLS/WTF

These days SSL has been replaced by TLS, but I'll use the terms interchangeably as people still refer to SSL "certificates" and TLS "connections". The way TLS works is rather fascinating, but I'm not going to dig into the detail of that as there are plenty of good explanations out there like [this one at Cloudflare](https://www.cloudflare.com/learning/ssl/transport-layer-security-tls/){:target="_blank"}. The main takeaway is that when a site has implemented TLS, the URL starts http**s** and users generally see some form of padlock icon in the address bar of their browser.

In the olden days, if you wanted a SSL certificate, you needed to contact a Certificate Authority (CA), and provide details of your domain name (and proof of ownership), company or individual identity documents, a bag of gold and your first born child. Thankfully, as "privacy and security" became seen more as something everyone should have by rights rather than something you get if you can afford it (a bit like healthcare - depending on where you're reading this from), the process became much more simple. Don't get me wrong, you can still pay a decent sum for a top-of-the-range, extended validation, premium certificate, but equally, thanks to some groundbreaking work by a non-profit CA called [Let's Encrypt](https://letsencrypt.org/){:_target="_blank"}, you can also get one for free*.

Let's Encrypt have a stated mission of creating "a more secure and privacy-respecting Web". Back at the end of 2015, it started issuing SSL certificates to websites for free. With that cat out of the bag, others followed suit and now you don't have to look far to find a basic SSL certificate without parting with your hard earned cash, for example from your hosting provider, Cloudflare, ZeroSSL and of course Let's Encrypt (now allegedly the largest CA on the planet).

## Setting up Let's Encrypt


Obtaining a SSL certificate from Let's Encrypt is incredibly simple. I took their advice and used certbot along with [their instructions for Apache and snap](https://certbot.eff.org/instructions?ws=apache&os=snap). I won't list these verbatim, but I will say that I opted for the 'lazy' option of having certbot automatically reconfigure my Apache install to setup HTTPS using `sudo certbot --apache` and it **just worked**. Nice.

One thing I didn't originally do was test the automatic renewal of the cert. This broke things later on, as during the initial setup, once I saw that TLS was working well, I figured I could firewall off TCP/80 as it wasn't necessary. Unfortunately, certbot needs this for renewal and I only realised this when the Let's Encrypt certificate expired and didn't automatically renew. You can test to see if automatic renewal is working using the following command:

```
sudo certbot renew --dry-run
```

I didn't do this at first but it was useful to help me diagnose the problem when the cert expired.

So now we have a web server which is serving pages under TLS. Next we need to install a CalDAV server.



# Baikal

We touched upon the CalDAV protocol earlier in the post. There are a few open-source implementations of CalDAV servers such as [DAViCal](https://www.davical.org/){:target="_blank"} and [Radicale](https://radicale.org/v3.html){:target="_blank"}. However, after a little bit of back and forth I settled on Baikal, [an open-source CalDAV and CardDAV server](https://github.com/sabre-io/Baikal){:target="_blank"}, which appears to support more of the specification than the others and be under active development. Regardless, it appeared to provide the functionality that I was after so I decided to follow the [Apache instructions](https://sabre.io/baikal/install/) on the sabre.io website and again, it worked immediately which is a good sign.

Once Baikal is up and running you can browse to the main page, login with the admin user that you set up during initial install and under 'Users and resources' you can start adding members as shown below.

{% include image-link.html image_path="/assets/images/2025/08/11/baikal_add_user.png" alt_text="Adding a user to Baikal"  caption="Adding a new user to Baikal" width="75%" %}


The username and password are important as we'll use these in the next section to add each user's calendar to various client applications.


# Calendar Applications


In our household we run a mix of Operating Systems. I've found that [Evolution](https://help.gnome.org/users/evolution/stable/){:target="_blank"} works reasonably well for Linux, and I've seen various threads regarding Windows which suggest [Thunderbird](https://www.thunderbird.net/en-US/download/){:target="_blank"} is *OK* where there are a small number of users. Equally I've seen suggestions that both [AgenDAV](https://github.com/agendav/agendav){:target="_blank"} and the [Outlook CalDav Synchronizer](https://caldavsynchronizer.org){:target="_blank"} work well on Windows, but I must confess I haven't had a chance to try these.

In fact, our main requirement was an Android client, as we primarily manage our calendars from our phones. The interesting thing about this is you need to install two applications: [DAVx<sup>5</sup>](https://www.davx5.com/){:target="_blank"}, a service which performs the *synchronisation* of your contacts, calendars and tasks and a CalDAV client application, which displays your calendar entries.

One thing to note about DAVx<sup>5</sup>, is that if you install it from Google Play it will cost you ~£5, whereas if you install F-Droid (the Free and Open Source Android App Repository), you can get it for free!

{% include image-link.html image_path="/assets/images/2025/08/11/davx5_play_store.png" alt_text="Screenshot of DAVx5 for £4.99 on Google Play"  caption="Only £4.99 from Google Play Store!" width="75%" %} 

## DAVx5 Setup

Once you've installed DAVx<sup>5</sup>, you need to select the '+' button to add a new account. You'll first be prompted to select a login type, for which you need to select 'URL and user name' in the options below:

{% include image-link.html image_path="/assets/images/2025/08/11/davx5_add_account.png" alt_text="Screenshot of adding a new account"  caption="Adding a new account" width="50%" %} 

In the next screen you'll be prompted to enter the URL of you CalDAV server (e.g. `https://dav.myawesomedomain.xyz/dav.php`), the user name of one of your accounts and that user's password.

{% include image-link.html image_path="/assets/images/2025/08/11/davx5_url_username.png" alt_text="Screenshot of adding URL and user name login"  caption="URL and username login screen" width="50%" %} 

Once you've added all your users your main DAVx<sup>5</sup> screen should look a little something like this:


{% include image-link.html image_path="/assets/images/2025/08/11/davx5_main_screen.png" alt_text="Screenshot showing accounts"  caption="Accounts are added and in sync" width="50%" %} 


## Etar Setup

I chose to use [Etar](https://github.com/Etar-Group/Etar-Calendar){:target="_blank"} for our Android client. It is a reasonably 'bare-bones' Calendar implementation, that the author says is an "enhanced version of the AOSP Calendar", but most importantly it's open-source. You can download pre-built versions on both Google Play Store and, of course, F-Droid for free. As we've already installed and configured DAVx<sup>5</sup>, it will automatically display those calendar events, but you can also add new calendars from within 'Settings' by selecting 'Add CalDAV calendar', which will show you the DAVx<sup>5</sup> screen we saw before.

Finally, Etar also supports Android 'Widgets' which means if you go to a blank area on your Android home screen and long press it you'll be prompted with a menu which includes 'Widgets'. Select this and scroll down to 'Etar' and you'll see you can add a nice 2x3 widget to display your upcoming events live. Awesome!


{% include image-link.html image_path="/assets/images/2025/08/11/etar_widget.png" alt_text="Screenshot of Etar widget screen"  caption="I was overly excited by this feature" width="50%" %} 


That's it for this post - I hope it was useful or interesting (or ideally both!).


\* *The same caveat at the start of this post applies. It doesn't cost you money, but the trade offs are that it needs to be revalidated every 90 days and there's no dedicated technical support, but if you're tech savvy enough this shouldn't be an issue.*
