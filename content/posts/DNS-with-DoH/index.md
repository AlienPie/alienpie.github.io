---
title: Configuring local DNS with DoH and Ad-Blocking
description: I decide to explore the world of DNS and host my own DNS service, with encrpytion and ad-blocking by default.
author: "Stan Jewhurst"
date: 2020-03-09T11:37:22+00:00
url: /local-dns-with-doh-and-ad-blocking/
cover:
  image: images/cover-image.jpg
  #caption: A featured image!
tags:
  - cloudflared
  - dns
  - doh
  - power-dns
  - privacy
ShowToc: false
draft: false
---
As the license on my Meraki Cloud gear gets closer to expiration I've been working more and more to get my home environment migrated to some new technologies. Some of these focus on simplification of my media set up - moving services like Plex back from the cloud and into an On Prem environment. Others focus on implementing some new tech to make managing my infrastructure more simple.

Once you start to get past a couple of services, running a DNS server starts to become quite attractive. It allows you to more easily manage your environment, and makes accessing servers and services much more straightforward - especially for mobile devices where typing an IP address can be a pain.

I've used PiHole running on Debian VM for a long time to block adverts at home, adding entries manually for local devices as I've needed them but I wanted something which allowed me to manage local entries more simply. I also wanted to implement DoH (DNS over HTTPS) given the fact that privacy is increasingly coming [under attack][1]. DoH encrypts DNS traffic between you and the Internet DNS Server, ensuring that eyes in the middle (such as ISPs) cannot [store, datamine or sell this information][2]. However, the provider at the far end still _can_, and there have been a lot of concerns about data retention and sale.

But where to begin? I've a little experience with PowerDNS, so decided it would be the best choice for the majority of my DNS efforts. I knew I would need two parts - an Authoritative DNS serer to create and manage my local zones, and a recursor to resolve internet hosts and insure that the local hosts were forwarded to the authoritative server. PowerDNS is modular, with the recursor and the Authoritative parts running separately. You can also chose the backend storage methodology from a wide range - I went for a local database as it is also supported by PDNS Manager - the web management interface I chose to manage the Authoritative server. It has a clean, easy to use interface and seemed straightforward to intall. For DoH, Cloudflare's cloudflared was my choice, as Cloudflare currently conform to [Mozilla's strict policies][3] on DNS DoH Providers.

There were a few order of operations issues I came across, but in the end, the following process was followed. Note that I already had PiHole installed on another server:

  * Install PDNS Authoritative and Recursive servers
  * Install MariaDB and PDNS MariaDB backend
  * Install PDNS Manager and Web Server (I like `apache`)
  * Download and install `cloudflared`
  * Configure PDNS Manager + PDNS Authoritative Server
  * Configure `cloudflared`
  * Configure PDNS Recursor
  * Configure PiHole

This seems like a lot but it's pretty straight forward - Get the packages:

```bash
stadmin@ns~# apt install pdns-server pdns-recursor pdns-backend-mysql mariadb-server apache2 php7.3 php7.3-json php7.3-mysql php7.3-readline php-apcu
```

Get and prepare PDNS Manager:

```bash
stadmin@ns~# wget https://dl.pdnsmanager.org/pdnsmanager-2.0.1.tar.gz
stadmin@ns~# tar -zxf pdnsmanager-2.0.1.tar.gz
stadmin@ns~# mv pdnsmanager-2.0.1/* /var/www/html
stadmin@ns~# chown -R www-data:www-data /var/www/html/frontend
```

And `cloudflared`:

```bash
stadmin@ns~# wget https://bin.equinox.io/c/VdrWdbjqyF/cloudflared-stable-linux-amd64.deb
stadmin@ns~# dpkg -i cloudflared-stable-linux-amd64.deb
```

Let's start configuration with MariaDB - secure the database by running the built in script:

```bash
stadmin@ns~# mysql_secure_installation
```

Then create a new empty database to use:

```bash
stadmin@ns~# mysql -u root -p
Enter password:

MariaDB [(none)]> CREATE DATABASE powerdns;
Query OK, 1 row affected (0.001 sec)
MariaDB [(none)]> exit
Bye
```

Next, configure Apache, PHP and PDNS Manager. I'm using the configuration available in the [PDNS Manager Docs][4]:

```bash
stadmin@ns~# a2ensite default-ssl
stadmin@ns~# a2enmod rewrite ssl php7.3
stadmin@ns~# phpenmod apcu json readline
stadmin@ns~# systemctl restart apache2.service
```

You should now be able to browse to `https://<YOUR-SERVER>/setup` and set up your database connection and user credentials. You'll get an invalid cert warning unless you've properly configured SSL.

With that done, we now configure `pdns-server` with the same information. Note that I only configure it to listen locally and on port 5300. This is because `pdns-recursor` and PDNS Manager will be the only services using it.

```bash
stadmin@ns~# vim /etc/powerdns/pdns.conf

#--------
#SETTINGS
#--------
allow-axfr-ips=127.0.0.1
allow-dnsupdate-from=127.0.0.1/8,::1
config-dir=/etc/powerdns
daemon=yes
disable-axfr=no
local-address=127.0.0.1
local-port=5300
#--------
#DATABASE
#--------
launch=gmysql
gmysql-host=127.0.0.1
gmysql-port=3306
gmysql-dbname=powerdns
gmysql-user=root
gmysql-password=<PASSOWRD>

stadmin@ns~# systemctl restart pdns.service
```

And finally enable it to run at boot:

```bash
stadmin@ns~# systemctl enable pdns.service
```

Now you can log into PDNS Manager and add your Zones and Records.

Next up, we configure `cloudflared`. `cloudflared` will use default settings, running on port 53 unless otherwise configured. This can be done with a short YAML file located in `/etc/cloudflared/config.yml`:

```yaml
proxy-dns: true
proxy-dns-port: 5053
proxy-dns-upstream:
	- https://1.1.1.1/dns-query
	- https://1.0.0.1/dns-query
```

Enable and start cloudflared:

```bash
stadmin@ns~# systemctl enable cloudflared.service
stadmin@ns~# systemctl start cloudflared.service
```

You can confirm it's running with `netstat` or by trying to resolve a domain:

```bash
stadmin@ns~# dig +short @localhost -p 5053 google.com
216.58.213.110
```

Almost there - onto the PDNS Recursor now!

```bash
stadmin@ns~# vim /etc/powerdns/recursor.conf

allow-from=127.0.0.0/8,<LOCAL-NETWORK>
forward-zones=<YOUR-ZONE>=127.0.0.1:5300
forward-zones-recurse=.=127.0.0.1:5053
local-address=0.0.0.0

stadmin@ns~# systemctl restart pdns-recursor.service
stadmin@ns~# systemctl enable pdns-recursor.service
```

Again confirm with `netstat` and `dig` (Note, no port needed this time):

```bash
stadmin@ns~# dig +short @localhost bbc.com
151.101.0.81
151.101.192.81
151.101.64.81
151.101.128.81
```

Our DNS server is now answering requests on port 53, and forwarding anything for the internet to `cloudflared` using DoH. For local zones defined in the `recursor.conf` file, it will forward them to the PDNS Authoritative Server:

```bash
stadmin@ns~# dig +short @localhost pihole.jewhurst.net
172.16.25.25
```

While this setup will work entirely on it's own, I also wanted to enabled ad-blocking with PiHole - so the final step is adding our newly configured DoH DNS Server to PiHole as an upstream server. It should be the only server selected unless you create a second setup as above. Note that you'd need to enable MySQL replication for the Authoritative Server.

Ensure PiHole is your DHCP Server, or your Router hands out PiHoles IP as the DNS server of choice and the setup is complete. You can even go as far as blocking port 53 on your firewall. 

So there you go - fully secured DNS over HTTPS with an internal authoritative name-server and added ad-blocking.

 [1]: https://www.libertyhumanrights.org.uk/human-rights/privacy/snoopers-charter
 [2]: https://blog.benjojo.co.uk/post/ISPs-sharing-DNS-query-data
 [3]: https://wiki.mozilla.org/Security/DOH-resolver-policy
 [4]: https://pdnsmanager.org/quickstart/