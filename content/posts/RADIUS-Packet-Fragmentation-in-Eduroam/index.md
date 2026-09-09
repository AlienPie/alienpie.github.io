---
title: "Eduroam, EAP-TLS and a Dropped Packet Fragment"
tags:
- Networking
- eduroam
- RADIUS
- Troubleshooting
- CERN
description: Students from a visiting school could not get onto eduroam using certificates. The cause turned out to be an ACL in our own network quietly dropping the non-initial fragments of large RADIUS packets.
date: '2026-09-09T14:58:15+02:00'
# weight: 1
# aliases: ["/first"]
author: "Stan Jewhurst"
showToc: true
TocOpen: true
draft: false
hidemeta: false
comments: false
#url: "/page-url/"
disableShare: false
hideSummary: false
searchHidden: false
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
cover:
    image: "images/cover.png" # image path/url
    alt: "A censored packet capture showing primary packet fragments but no secondary part." # alt text
#    caption: "<text>" # display caption under cover
#    hidden: true # only hide on current single page
---
Eduroam is one of the services I took on when I joined CERN. It's the thing that lets someone visiting from another university open their laptop, pick the `eduroam` SSID, and get online with their home account without anyone having to hand them a guest login. When it works, which is almost always, nobody gives it a second thought.

This is about a case where it didn't. A group of students from a secondary school in Italy were due to come and visit, and on every previous visit they had been unable to connect. It had been raised, looked at, and shelved several times over about two years - each time the students went home before anyone got to the bottom of it, and the ticket went quiet again. This time I had an expired test certificate from the school to work with, and a contact at [SWITCH](https://www.switch.ch/), the Swiss NREN (National Research and Education Network), who was willing to help from their side.

TL;DR: an ACL in our own network was letting the first fragment of a fragmented RADIUS packet through and silently dropping everything after it. Certificate authentication produces packets big enough to fragment and password authentication doesn't, which is why one worked and the other didn't.

IPs, internal hostnames and the school's name are all placeholders below.

---
## How an eduroam login gets to where it's going

If you already know how eduroam roaming works you can skip this bit. If you've only ever used it, here's the shape of it:

Eduroam is a large federated RADIUS network. When you authenticate at somewhere you're visiting, your credentials aren't checked there - the visited site has no idea who you are or what your password is. Instead it looks at your realm (the `@school.example.it` part of your identity), works out that it isn't local, and proxies the request off towards your home institution, which is the only party that can actually authenticate you.

At CERN we run the Service Provider (SP) end of that - the RADIUS servers our access points talk to. Anything that isn't a `cern.ch` realm gets handed to SWITCH. From there it goes up to the European top-level servers, across to GARR (the Italian NREN), and finally to the school itself. The `Access-Accept` or `Access-Reject` comes back down the same chain of proxies.

```
  your device
      |          EAP over 802.1X
  CERN SP                <- us
      |          RADIUS over UDP, routed on the realm
  SWITCH  (Swiss NREN)
      |
  eduroam European top-level servers
      |
  GARR  (Italian NREN)
      |
  the school's IdP       <- actually checks the credentials
```

Two things about this matter for the rest of this post. 

The first is that RADIUS between proxies runs over UDP, historically on port 1812. There's no session and no concept of retransmitting part of a message - if a packet doesn't arrive intact, the whole request just times out, and the proxy that sent it eventually decides the next server along is dead.

The second is that certificate authentication produces much larger packets than password authentication. A username and password is a handful of bytes. EAP-TLS runs a full TLS handshake inside the EAP exchange, certificates and all, and the server's half of that - its certificate, any intermediates, the key exchange - is big enough that the RADIUS packet carrying it can go over the usual 1500 byte MTU and end up fragmented at the IP layer.

There's a wrinkle here that cost me the best part of a day. RADIUS has its own mechanism for moving large EAP payloads around: the server hands over the TLS data a slice at a time, one slice per `Access-Challenge`, and the client acknowledges each slice before the next is sent. That's EAP fragmentation, and it's the long run of `Request, TLS EAP` / `Response, TLS EAP` frames you see in a capture of any EAP-TLS handshake.

The catch is that EAP fragmentation only happens on the server that's terminating the TLS session. A proxy in the middle - which is all CERN is here - doesn't reassemble the EAP payload and re-slice it, it just forwards the RADIUS attributes it was given. So if one of those forwarded packets is over the MTU, it's down to ordinary IP fragmentation to get it there in one piece, and IP fragmentation is far less reliable than most people assume.

---

## Password works, certificate doesn't

There were no live authentication attempts from the school's realm to look at - the ticket had been raised ahead of the visit - so the first thing to do was reproduce it. I set up a local supplicant with the expired certificate and ran two test authentications against the school's realm: one with a deliberately wrong password, one with the certificate.

The password attempt failed, but it failed in the right way:

```
(1) Received Access-Reject Id 160 from <FTLR1>:1812 to <CERN-SP>:37304 length 63
(1)   Reply-Message = "Request Denied"
(1) Login incorrect (Home Server says so): [eduroam-test@school.example.it]
```

`Home Server says so`. The request went all the way to Italy, the school's IdP turned down the bad password, and the rejection came all the way back. The federation path is fine.

The certificate attempt failed differently:

```
(5) ERROR: Failing proxied request for user "eduroam-test@school.example.it", due to lack of any response from home server <FTLR1> port 1812
(5) Login incorrect (Home Server failed to respond): [eduroam-test@school.example.it]
```

`Home Server failed to respond`, and on our own RADIUS server the giveaway line:

```
Proxy: Marking home server <FTLR1> port 1812 as zombie (it has not responded in 20.000000 seconds).
```

Same user, same realm, same route, tested within a minute of each other. The small packets get an answer and the large ones disappear. That matched the notes from the previous rounds of this ticket, where a colleague had seen exactly the same split.

So: a size problem, on a path that runs from Geneva to Italy and back. Not a small area to search.

---

## Ruling things out

SWITCH pointed at two things worth eliminating before going any further.

**Were we doing dynamic discovery?** Modern eduroam can skip the proxy chain: the SP looks up a `NAPTR` record for the realm and, if there is one, connects directly to the home institution's RadSec (RADIUS/TLS) endpoint. The realm does have one:

```
$ dig school.example.it NAPTR
school.example.it.  IN  NAPTR  100 10 "S" "x-eduroam:radius.tls" "" _radsec._tcp.eduroam.it.

$ dig _radsec._tcp.eduroam.it SRV
_radsec._tcp.eduroam.it.  IN  SRV  10 0 2083 radius2.garr.net.
_radsec._tcp.eduroam.it.  IN  SRV  20 0 2083 radius.garr.net.
```

If our servers were doing that lookup and then failing to bring up the direct connection, that would fit the symptoms. They aren't:

```
$ grep -r "use_naptr" /etc/raddb/
$                       # no output - dynamic discovery is not enabled
```

Our `proxy.conf` sends `cern.ch` to the local server and everything else to the SWITCH pool. No dynamic lookups. Ruled out.

**Was the certificate chain too big for RADIUS?** A RADIUS packet can't be larger than 4096 octets, full stop. A large enough certificate chain would break that limit and produce a malformed packet rather than a fragmented one. The test certificate's chain was a leaf and one intermediate - nowhere near it. Ruled out.

**Does fragmentation work on this path at all?** This is the one that nearly sent me off in the wrong direction. Our network runs a 1500 byte MTU and fragmentation looked completely healthy:

```
$ ping -s 1472 -M do <FTLR1>          # 1500 bytes on the wire, don't fragment
1480 bytes from <FTLR1>: icmp_seq=1 ttl=51 time=5.48 ms

$ ping -s 9000 -M dont <FTLR1>        # 9028 bytes, allow fragmentation
9008 bytes from <FTLR1>: icmp_seq=1 ttl=51 time=6.12 ms
```

A 9000 byte ping - six fragments - to SWITCH's server and back in 6ms. As far as ICMP was concerned there was nothing wrong with fragmentation on this path. That turned out to be worth nothing at all, but I didn't know it yet.

---

## A packet that fragments on demand

The thing that actually moved this forward was a test packet SWITCH sent over. It's a RADIUS `Status-Server` request padded out with a stack of `Proxy-State` attributes until it's certain to be over the MTU. `Proxy-State` is useful here because the RADIUS spec says a server has to echo every one back untouched, so a single request and its reply put a large packet on the wire in both directions.

```
echo "Message-Authenticator = 0x00, \
  Proxy-State = 'this-is-SWITCH-testing-for-fragmentation-issues-on-UDP-they-are-truly-annoying-and-affecting-EAP-authentication-in-RADIUS-quite-a-bit', \
  Proxy-State = 'sometimes-fragmentation-issues-show-up-sometimes-they-dont-it-may-depend-on-the-RADIUS-software-used-or-the-EAP-type-itself', \
  Proxy-State = 'for-instance-EAP-TLS-has-a-high-payload-originating-from-the-supplicant-causing-larger-packets-to-be-fragmented', \
  Proxy-State = '...and so on, another dozen of these to push it well over 1500 bytes...'" \
  | radclient -t 2 -r 1 -x <FTLR1>:1812 status '<shared-secret>'
```

Same 20 second timeout as the certificate authentication. No reply, no echo, nothing.

That took eduroam, EAP and certificates out of the picture. What was left was much simpler: a large UDP packet from our RADIUS server to SWITCH doesn't arrive, and I could now cause that with one command rather than driving a whole EAP-TLS handshake each time.

---

## On a packet capture

With a one-line reproducer I could capture at both ends of the path and compare. I picked two points: the QA RADIUS server's own interface, and our external firewall, which is the last CERN device the traffic passes before it leaves for SWITCH.

### Leaving the RADIUS server

```
No.  Time       Source   Destination  Proto   Len   Info
 1   0.000000   CERN-SP  FTLR1        RADIUS  239   Response, Identity
 2   0.091603   FTLR1    CERN-SP      RADIUS  109   Request, TLS EAP (EAP-TLS)
 3   0.093399   CERN-SP  FTLR1        RADIUS  510   Client Hello
 4   0.134959   FTLR1    CERN-SP      RADIUS  1135  Request, TLS EAP (EAP-TLS)
 5   0.135766   CERN-SP  FTLR1        RADIUS  223   Response, TLS EAP (EAP-TLS)
 6   0.179792   FTLR1    CERN-SP      RADIUS  1135  Request, TLS EAP (EAP-TLS)
 7   0.180772   CERN-SP  FTLR1        RADIUS  223   Response, TLS EAP (EAP-TLS)
 8   0.228454   FTLR1    CERN-SP      RADIUS  1135  Request, TLS EAP (EAP-TLS)
 9   0.229206   CERN-SP  FTLR1        RADIUS  223   Response, TLS EAP (EAP-TLS)
10   0.272691   FTLR1    CERN-SP      RADIUS  780   Server Hello, Certificate, Server Key Exchange, Certificate Request, Server Hello Done
11   0.278662   CERN-SP  FTLR1        RADIUS  1635  Response, TLS EAP (EAP-TLS)
12   3.278936   CERN-SP  FTLR1        RADIUS  1635  Response, TLS EAP (EAP-TLS)
13   9.279371   CERN-SP  FTLR1        RADIUS  1635  Response, TLS EAP (EAP-TLS)
14  21.279833   CERN-SP  FTLR1        RADIUS  1635  Response, TLS EAP (EAP-TLS)
15  41.284379   CERN-SP  FTLR1        RADIUS  112   Status-Server id=133
16  41.289539   FTLR1    CERN-SP      RADIUS   80   Access-Accept id=133
```

The handshake runs normally through the small exchanges. Packet 11 is a 1635 byte RADIUS packet carrying the client's reply to the server's certificate - over the MTU, so it leaves the server as two IP fragments.

Then nothing comes back. The server retransmits the same 1635 byte packet at 3, 9 and 21 seconds (packets 12 to 14), gives up, and at 41 seconds sends a small `Status-Server` health check instead (packet 15). That gets an immediate `Access-Accept` (packet 16). The server upstream is alive and reachable - it just never saw the big packet.

### Arriving at the firewall

```
No.  Time       Source   Destination  Proto  Len   Info
 1   0.000000   CERN-SP  FTLR1        RADIUS 243   Response, Identity
 2   0.091050   FTLR1    CERN-SP      RADIUS 113   Request, TLS EAP (EAP-TLS)
 3   0.093393   CERN-SP  FTLR1        RADIUS 514   Client Hello
 4   0.134523   FTLR1    CERN-SP      RADIUS 1139  Request, TLS EAP (EAP-TLS)
 5   0.135756   CERN-SP  FTLR1        RADIUS 227   Response, TLS EAP (EAP-TLS)
 6   0.179366   FTLR1    CERN-SP      RADIUS 1139  Request, TLS EAP (EAP-TLS)
 7   0.180759   CERN-SP  FTLR1        RADIUS 227   Response, TLS EAP (EAP-TLS)
 8   0.227960   FTLR1    CERN-SP      RADIUS 1139  Request, TLS EAP (EAP-TLS)
 9   0.229199   CERN-SP  FTLR1        RADIUS 227   Response, TLS EAP (EAP-TLS)
10   0.272257   FTLR1    CERN-SP      RADIUS 784   Server Hello, Certificate, Server Key Exchange, Certificate Request, Server Hello Done
11   0.278963   CERN-SP  FTLR1        IPv4   1518  Fragmented IP protocol (proto=UDP 17, off=0, ID=787d)
12   3.279263   CERN-SP  FTLR1        IPv4   1518  Fragmented IP protocol (proto=UDP 17, off=0, ID=7e69)
13   9.279759   CERN-SP  FTLR1        IPv4   1518  Fragmented IP protocol (proto=UDP 17, off=0, ID=85bd)
14  21.280161   CERN-SP  FTLR1        IPv4   1518  Fragmented IP protocol (proto=UDP 17, off=0, ID=9a54)
```

Same test, one hop further along. Identical up to packet 10, and then, in place of a decodable 1635 byte RADIUS packet, the firewall sees this:

```
Fragmented IP protocol (proto=UDP 17, off=0, ID=787d)
```

`off=0` is the first fragment, and it's the only fragment that arrives. Wireshark can't decode it as RADIUS because the second fragment never turns up. Every retransmission comes through the same way - one first fragment, a fresh IP ID, nothing after it. Something between the RADIUS server and the firewall was passing fragment one and dropping the rest.

### The ACL

My colleague and I walked the path and found a router with an ACL on it - we use these a lot, so it wasn't that surprising. However it was only matching on protocol (UDP) and port (1812), and ACLs are stateless, so any packets _had_ to be an exact match.

Only the first fragment of a fragmented IP packet carries the Layer 4 header. Fragment one of ours had source and destination port 1812 on it, matched the permit rule and went through. Every fragment after it carries no port information at all, so it couldn't match a port-based rule, hit the implicit deny at the end of the list, and was dropped.

This is a well-known way for stateless packet filters and fragmented UDP to go wrong, and RADIUS with EAP-TLS is close to an ideal trigger for it - everything works fine for people using passwords, so most of the users have no issues. It's only more recently that certificates are starting to become more common for eduroam deployments (~20%). 

This also explained why CERN certificate continued to work on remote sites - the ACL was only in one direction, and so didn't affect incoming eduroam authentications from abroad.

### The fix

The change was small - an ACL entry permitting IP traffic (all of it, not just UDP/1812) between the QA RADIUS server and SWITCH's server, so that fragments with no port information still match on address alone.

Same reproducer, same capture point, after the change:

```
No.  Time      Source   Destination  Proto  Len   Info
 1   0.000000  CERN-SP  FTLR1        RADIUS 243   Response, Identity
 2   0.058856  FTLR1    CERN-SP      RADIUS 113   Request, TLS EAP (EAP-TLS)
 3   0.061266  CERN-SP  FTLR1        RADIUS 514   Client Hello
 4   0.110684  FTLR1    CERN-SP      RADIUS 1139  Request, TLS EAP (EAP-TLS)
 5   0.112285  CERN-SP  FTLR1        RADIUS 227   Response, TLS EAP (EAP-TLS)
 6   0.162849  FTLR1    CERN-SP      RADIUS 1139  Request, TLS EAP (EAP-TLS)
 7   0.164506  CERN-SP  FTLR1        RADIUS 227   Response, TLS EAP (EAP-TLS)
 8   0.219533  FTLR1    CERN-SP      RADIUS 1139  Request, TLS EAP (EAP-TLS)
 9   0.221169  CERN-SP  FTLR1        RADIUS 227   Response, TLS EAP (EAP-TLS)
10   0.264564  FTLR1    CERN-SP      RADIUS 784   Server Hello, Certificate, Server Key Exchange, Certificate Request, Server Hello Done
11   0.271370  CERN-SP  FTLR1        IPv4   1518  Fragmented IP protocol (proto=UDP 17, off=0, ID=cd35) [Reassembled in #12]
12   0.271367  CERN-SP  FTLR1        RADIUS 159   Response, TLS EAP (EAP-TLS)
13   0.271573  CERN-SP  FTLR1        IPv4   1518  Fragmented IP protocol (proto=UDP 17, off=0, ID=cd35) [Reassembled in #14]
14   0.271578  CERN-SP  FTLR1        RADIUS 159   Response, TLS EAP (EAP-TLS)
15   0.318409  FTLR1    CERN-SP      RADIUS 113   Request, TLS EAP (EAP-TLS)
16   0.320060  CERN-SP  FTLR1        RADIUS 760   Certificate, Client Key Exchange, Certificate Verify, Change Cipher Spec, Encrypted Handshake Message
17   0.379062  FTLR1    CERN-SP      RADIUS 109   Failure
```

The fragments arrive together now. Wireshark reassembles the 1635 byte packet (`[Reassembled in #12]`) and decodes the RADIUS layer again (packet 12). The handshake carries on from where it had been stalling: the home server asks for the client's half of the exchange (packet 15), the client sends its certificate and key exchange (packet 16), and it finishes - at 0.38 seconds rather than never - with an EAP `Failure` (packet 17).

The `Failure` is the right outcome: the test certificate is expired, so the school is correct to turn it down. What matters is that the authentication ran all the way to the end. A real student with a valid certificate gets an `Access-Accept` there instead.

The same change then went onto the production eduroam servers.

---

The part of this I'll remember is how convincingly the ping test lied. A 9000 byte ICMP echo crossed the path without any trouble, and if I'd taken that at face value I'd have ruled out the network early and spent a lot longer than I did reading through RADIUS configs. Fragmented UDP to a particular port on a particular host is its own test, and an ICMP echo to the same host isn't a substitute for it.

The proper fix for this whole class of problem is RadSec - RADIUS over TLS, on TCP port 2083 - where TCP handles segmentation and there are no UDP fragments to lose in the first place. A good part of the eduroam infrastructure already runs on it, and the more of the path that does, the less room there is for something like this. That's a larger piece of work though, and the ACL change sorted out the immediate problem.

Another great result of this is that we saw a 90% drop in eduroam authentication failures for vistors and collaborators from other institutions whose eduroam had been silently failing for all this time!

