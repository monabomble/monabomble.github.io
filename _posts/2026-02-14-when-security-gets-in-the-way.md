---
layout: post
title:  "When security gets in the way"
subtitle: "The curious case of AVG AntiTrack"
date: 2026-02-14 15:00:00
background: '/assets/images/2026/02/14/fingerprint.png'
---

# Background

I was visiting my mum a few weeks back. Normally I take the kids, but this time I wanted to spend some quality time just <s>being waited on</s> hanging out with the old girl. Fortunately, my mum knows that I am unable to sit completely idle for too long and also that I do love a technical puzzle. She mentioned that a few of the websites she commonly visits were refusing to load with an error in the five hundred range (despite being in her 70s she has a background in I.T. so notices the details).

My first thought was some sort of server error. Of course we all know that HTTP 5xx codes mean some sort of server error right? I mean, "RFC7231 Hypertext Transfer Protocol (HTTP/1.1): Semantics and Content" is quite explicit:

<blockquote>
"The 5xx (Server Error) class of status code indicates that the server
   is aware that it has erred or is incapable of performing the
   requested method"
</blockquote>

That said, my mum mentioned multiple well known websites, such as those hosted on UK Government domains and airlines, were affected. I hadn't heard anything about any widespread problems recently, so I decided to take a look.

I should say that our family grew up with various variants of the trusty Macintosh and, as old habits die hard my mum still prefers an iMac to any other sort of home PC setup. She logged in and fired up Safari and tried to visit the website for the [UK's General Register Office](https://www.gro.gov.uk/gro/content/), which she uses for one of her hobbies involving researching her ancestry.

{% include image-link.html image_id="bad_gateway" image_path="/assets/images/2026/02/14/bad_gateway.png" alt_text="A 502 Bad Gateway error" link="#" caption="Well this doesn't look right" width="75%" %}

Bad gateway? Referring back to RFC7231, this "indicates that the server, while acting as a gateway or proxy, received an invalid response from an inbound server it accessed while attempting to fulfill the request". Well, that's cleared that up then... &#128580;

Perhaps more interesting here is the subtext, which mentions 'Certificate verify failed'. In a previous blog, I went into some detail around Transport Layer Security (TLS). To avoid repeating that, I'll just say that each of the websites, for which my mum was seeing this error, was protected by TLS. This means her machine and the remote machine negotiate a set of security parameters to ensure that the connection is secure (any information exchanged is encrypted and the server really belongs to the organisation it claims to belong to). This error suggests that part of this process is not working as expected. The game is afoot! &#x1f575;

# Sniffing packets

Anyone with a background in networking or IT in general will be aware that, regardless of how two computers are connected (e.g. a cable, WiFi, satellite link, or [carrier pigeon](https://www.rfc-editor.org/rfc/rfc2549.html)) they need to use a standard "protocol" for communication. Regular users of applications or web browsers don't normally get to see this communication - it just works&#8482;

However, those in the industry have various special tools that allow us to peek behind the curtain and see the raw communication. In the case of web browsers, information is exchanged using several protocols (TLS, HTTP, TCP, IP, etc.). Obviously, we all want responsive/dynamic web pages and applications so that information tends to be 'chunked' so the user's web browser doesn't neccesarily need to receive all the information before it starts displaying it. The information chunks are knowns colloquially as "packets" and we can use a tool to collect and analyse these packets - known as a packet sniffer (if you know who came up with this term, do let me know!).

There are many different tools to do this but by far the most well known is Wireshark, a cross-platform, open-source packet sniffer. Whilst just collecting the raw bytes from packets flying across the wire/the air is useful, far more useful is where a tool can accurately <i>decode</i> the packet. What do I mean by this? Well consider what a 'raw' packet might look like:

```
47 45 54 20 2f 20 48 54  54 50 2f 31 2e 31 0d 0a
48 6f 73 74 3a 20 77 77  77 2e 73 6f 6d 65 73 69
74 65 2e 63 6f 6d 0d 0a  0d 0a
```

This is just a sequence of hexadecimal (base 16) numbers, for which it is hard to instantly infer meaning without some context and technical knowledge. However, Wireshark will have some context to help decode this. For example, it might know that this packet was received on Transport Control Port (TCP) number 80. TCP/80 is well-known as the port for Hypertext Transfer Protocol or HTTP (used for serving web content). Much like the RFC document referenced above, there is a [set of rules](https://datatracker.ietf.org/doc/html/rfc2616) published on how to interpret bytes on this port. I won't walk through that process, but the upshot is that Wireshark will show the analyst this decoded view of the same bytes.

```
GET / HTTP/1.1
Host: www.somesite.com
```

This is much easier for a techie to identify. It is a request from a client to a server, asking for the default page at the root of the web server's page directory. It is using version 1.1 of the HTTP protocol and therefore the client has added a 'Host' header to specify which website they are interested in. This is because a lot of websites run on Virtual Private Servers (VPS) which host multiple sites on one 'machine', therefore we need to specific exactly which site we're interested in.


Ok, now we know what a packet sniffer is and why it is useful, it follows that someone like me might use one when we are trying to diagnose connection issues, especially where we have a hint that it is the <i>remote</i> end of the connection with the problem. Here's what I saw:

{% include image-link.html image_id="wireshark" image_path="/assets/images/2026/02/14/wireshark.png" alt_text="Partial output from Wireshark" link="#" caption="Sniff, sniff" width="100%" %}

If you aren't familiar with Wireshark you need to know that each line here represents a packet. We see a TCP handshake followed by a TLS handshake (the internet was founded on handshakes). In fact, it all looks very normal right up until packet number 18, where mum's Mac appears to randomly scream 'NOPE!' and walk away from the conversation... strange. This is NOT the server's problem.

In slightly more detail, the snippet shows a promising start. Mum's Mac (192.168.1.218) reaches out to gro.gov.uk (81.144.203.211), they have a bit of back and forth ("hi", "hello", "how're things?", "good, thanks", etc.). Then they get down to the business of exchanging ID (ok, the conversation analogy breaks down here unless you're living in a dictatorship I guess). In this case the server shared some ID and detail of the level of behaviour and decorum it expects from the convo, at which point mum's Mac seems to decide it's all too much.

# The enemy within

Remember that HTTP error we saw in the browser (502 Bad Gateway) and reasonably thought it was the server? That is definitely not the case. To send a HTTP error code, the server would have had to receive a HTTP request (like the GET request above). The Wireshark snippet above shows that we never get to that point. HTTP-based exchanges under a secure (TLS) connection can only occur when a secure 'tunnel' is established and the snippet shows us that the conversation is abruptly brought to a halt before that happens.

So if the error isn't coming from the server, where IS it coming from? Given the first clue pointed to a certificate problem I decided to start there. A certificate is a digital document which proves the identify of a website and can be used to set up the encryption used to protect communications. Whenever you visit a website which uses TLS (pretty much all of them these days), you can view the details of the certificate by clicking a 'padlock' icon which is normally located just to the left of the browser's address bar. I'm not that familiar with Macs and the Safari browser anymore so I had to look through the menu options to find it but here's what I saw when I examined the certifiate for https://gro.gov.uk:

{% include image-link.html image_id="odd_certificate" image_path="/assets/images/2026/02/14/odd_certificate.png" alt_text="Strange certificate details for gro.gov.uk" link="#" caption="I think we're about to find out whodunnit" width="75%" %}

There's a lot of detail here, but to the trained eye there are several red flags. Hell, to the untrained eye there's something disconcerting in this image. This is an official UK Government website, yet the issuer of the certificate appears to be based in the Czech Republic (CZ). Now, I'm not being xenophobic here. The majority of Western certificate authorities are based in the US, including Digicert who legitimately issue certificates to UK Government public bodies (so not even UK-based). However, I would say it would be <i>highly</i> irregular for a Czech based CA to be issuing UK Gov certs &#x1F928;.

There's a simple explanation though. The certificate for www.gro.gov.uk appears to have been issued by AVG, otherwise known as [AVG AntiVirus](https://en.wikipedia.org/wiki/AVG_AntiVirus). Ah-ha! This seems like the start of an explanation.

A quick check with mum and it seems that she has installed AVG and she thinks she might even be paying for something (AVG is often cited as a 'free' AntiVirus). A quick Command (\⌘) + N opens the 'Finder' window and typing AVG reveals the product in question.


{% include image-link.html image_id="antitrack" image_path="/assets/images/2026/02/14/anti_track.png" alt_text="AVG AntiTrack" link="#" caption="Avast!" width="75%" %}

AntiTrack is billed as a privacy tool that protects your identity by blocking tracking attempts and masking your digital fingerprint. That's a legitimate concern. A digital fingerprint is a collection of traits about your computer, web browser and network connection, which a server can determine when you connect to it. Just like a security guard who monitors CCTV might not immediately recognise all the people that she sees passing the camera on a daily basis, she can start to recognise invidivuals from their specific traits, such as red hair, a tattoo, a limp, etc. A web server or gateway can do something similar using your computer's unique properties. Why should you care? Well, this is a breach of your privacy as it allows advertisers to track you across the web without your consent, building a profile of you such as your income bracket, political affiliations and more. Creepy.

That said, privacy shouldn't come at the cost of preventing you from going about your business on the web. In this case it was, but why?

# Security Top Trumps

The way that AVG AntiTrack (and presumably several other products of this kind) work, is to intercept all communications between your browser and remote servers. It needs to do this, otherwise it would have no way to 'see' inside the encrypted tunnel your browser creates to ensure none of the computers (routers) between your computer and the web server can steal sensitive data (e.g. your internet banking password). The reason it needs to see all the data between you and the remote server is that it needs to replace your real browser and device details with randomised fake browser and device details, therefore impeding unwanted tracking.

{% include image-link.html image_id="cost" image_path="/assets/images/2026/02/14/cost.png" alt_text="AVG AntiTrack Annual Cost" link="#" caption="Yours for 50 quid a year" width="75%" %}

AntiTrack is £49.99 a year for one Mac so, given this is non-free, why isn't it working? Perhaps there has been some sort of update which has broken it? Let's check.


{% include image-link.html image_id="update" image_path="/assets/images/2026/02/14/update.png" alt_text="AVG AntiTrack Update Information" link="#" caption="Quiet in here..." width="75%" %}


Hmm, it actually hasn't been updated for over 18 months and checking for updates doesn't change this. That is surprising. The pace of browser technology changes reasonably quickly so, whilst there probably isn't quite a need to update this product as frequently as something like anti-virus definitions, I'd expect to see some activity. In fact, Safari gets updated monthly and Firefox is updated every two weeks, demonstrating that browser technologies are constantly changing.


And therein lies the problem. Whilst AVG AntiTrack is "legitimately" performing a machine-in-the-middle attack to be able to mitigate tracking data, the Safari browser is detecting some anomalies with the connection. This is much like when I saw the country code CZ and instantly felt something was wrong. With more than an hour or so at my mum's I might even have fired up a dissasembler to analyse exactly which bits of AVG's certificate didn't "smell right" to Safari, but I don't need to. The fact that the 502 error could not have come from the remote side, combined with the fact that it was mum's machine that dropped the connection is a fairly decent indicator.

The security feature of Apple's Safari browser is effectively breaking the privacy feature of AVG AntiTrack - amazing.


# Summary

A good question at this point is: does my mum *need* to continue paying for AVG AntiTrack. Well, let's look at how effective it is. This is fairly easy to do thanks to numerous websites which will test your 'tracking profile'. For example, with AVG AntiTrack turned on I used the excellent [Cover Your Tracks](https://coveryourtracks.eff.org) by the Electronic Frontier Foundation (EFF). This is a free service to test your browser's protection against tracking and fingerprinting. Here's the result:

{% include image-link.html image_id="protection" image_path="/assets/images/2026/02/14/strong_protection.png" alt_text="EFF Tool output showing strong protection" link="#" caption="The force is strong with this one" width="75%" %}


Amazing! Looks like AVG AntiTrack is doing a great job and the £50/year is totally worth it... or is it? Let's turn off Anti-Track by unchecking in its settings dialog by unchecking the box which says 'Enable HTTPS Filtering' and just to be sure we'll also uncheck the box that says 'Enable tracking protection and detection'.


And the difference? **None whatsoever**. The Cover Your Tracks result still reports that we have 'Strong Protection'. The reason for this is that Safari (and other browsers) already provide reasonably strong protection against tracking if you enable it. Check out the settings on my Mum's browser:


{% include image-link.html image_id="privacy" image_path="/assets/images/2026/02/14/privacy.png" alt_text="Safari settings to enable anti-tracking natively" link="#" caption="Built-in browser protections" width="75%" %}

It has built-in protection and this protection doesn't break on random websites and it doesn't come with a price tag. Happy days.



{% include note.html content="There is an [article on AVG's website](https://support.avg.com/SupportArticleView?l=en&urlName=Troubleshoot-AVG-AntiTrack-web-access-speed&supportType=home), which takes users through a number of troubleshooting steps when AVG AntiTrack \"blocks or slows down webpages\". The final step is just to turn it off. It doesn't mention asking for a refund or cancelling your subscription, which is exactly what I've advised my mum to do!" %}

