dns
===

Quickstart for pihole + unbound on the raspberry pi.

Mote info: https://www.youtube.com/watch?v=e_EfmKdP2ng

## Quickstart

- Set static ip for device (router to reserve upon new address request, r-pi to immediately request).

    - NOTE: In practice, r-pi dhcp setup will eliminate need for router-side dhcp reservation of r-pi address, _raspberry pi itself excepting_. Tldr; Router to reserve pihole ip, pihole to reserve everything else.

- Install pihole (interactive, will update packages for you).

    - NOTE: In interactive install, you may choose a temporary upstream dns provider, as this setting will be changed after you install unbound.

    - NOTE: Pay attention to the admin password the installer gives you (you may change this on the command line later).

- Test pihole dns requests / ad-blocking before network-wide rollout (say by visiting `googleadservices.com`).

    - HINT: You may set your machine to use the pihole's dns server directly (in Windows: Control Panel > View network connections...).

    - WARNING: The pihole must be the sole v4 dns provider AND the sole v6 dns provider on your network. While testing v4 resolution directly on your device (say, a Windows laptop), it's wise to disable v6 on your device's network interface to avoid being assigned a v6 dns provider by your router (we will revisit router configuration later to resolve this).

    - WARNING: Avoid cached results in chromium-based browsers by opening incognito sessions.

    - WARNING: If pihole is not your sole dns provider, Chromium-based browsers may use dns-over-https (DoT), which is more secure, but bypasses pihole. The solution, again, is to ensure pihole is the _SOLE_ v4 and v6 dns provider on your network. You may also manually disable DoT in Chrome under Settings: Privacy and Security > Security > Use secure DNS.

- Once you're happy that your pihole is doing what you want, it's time to undo any per-device changes and deploy the pihole to your network.

    - (In router) DISABLE dhcp + dns on your router and configure pihole as _SOLE_ v4 dhcp + dns server on your network.

    - (In router) DISABLE dhcp + dns on your router and configure pihole as _SOLE_ v6 dhcp + dns server on your network (requires auto config to be enabled on same advanced > v6 settings page). More info: https://discourse.pi-hole.net/t/how-to-get-rid-of-ghost-dns-servers/61953/9

    - NOTE: Secondary dns resolvers are not used as a backup in case the first one fails, they are used _simultaneously_ (and thus must be removed for pihole to do its job).

    - _(Might also have to enable v6 support in pihole settings?)_

- (Optional) Add firebog lists: `firebog.net`

    - NOTE: The lists with green checkmarks are generally safe and won't break websites.

    - NOTE: To get them in copy/paste format, navigate to the csv version link.

    - NOTE: To activate your changes, run `pihole -g` (as per web gui) and restart dns resolver in settings.

## (Optional) Configure Pi Hole for DNS Over TLS

_[NOTE: Edited for later versions of Ubuntu / Debian.]_

```bash

# One of the fundamental flaws of DNS is the lack of encryption or integrity, which allows your ISP to snoop DNS traffic or spoof a DNS response. DNS-over-TLS will not completely solve these problems, but it provides a step in the right direction.

# Pi-hole uses a fork of dnsmasq as it’s DNS server. To use DoT, we will actually need to run an additional DNS server, Unbound, that provides this feature. To install on a Debian-based system, run the following:

sudo apt install -y unbound dnsutils

# Once installed, run the following command to grab a configuration file:

sudo wget https://bartonbytes.com/pihole.txt -O /etc/unbound/unbound.conf.d/pihole.conf

# You’ll notice that this DNS server is configured to be accessible only on the local machine. It will open up port 5533. The config file includes the Quad9 and Cloudflare upstream DNS servers, which you can change or add to if necessary.

# Make sure that Unbound is running:

sudo systemctl restart unbound && sudo systemctl enable unbound

# To test that Unbound can fulfill your DNS requests, run the following dig command:

dig @127.0.0.1 example.com -p 5533

# Now, we need to tell Pi-hole’s dnsmasq to use this local port as it’s upstream DNS server. In the GUI, disable the upstream dns provider you selected during install and go to Settings -> DNS, and set a custom IPv4 server with the value 127.0.0.1#5533.

# Now we must restart Pi-hole:

sudo systemctl restart pihole-FTL

# ...and voila! The upstream DNS requests sent from your Pi-hole will be encrypted using TLS.

# As mentioned earlier, DNS-over-TLS is not a perfect solution to your privacy concerns. No matter how you protect your DNS traffic, the name of the websites that you visit will still be visible in the SNI of your HTTPS traffic, allowing your ISP (and any other intermediary) to view it. DoT somewhat protects integrity by preventing intermediaries from manipulating your DNS requests or their responses. However, you are still trusting the upstream DNS server- in our case, Quad9 and Cloudflare- to provide the correct responses.

# Another option to secure DNS traffic is DNS-over-HTTPS. I chose DoT because the cloudflared program would not work on my Raspberry Pi 1 Model B+. DoH has the advantage of being harder to block or detect, because the DNS traffic is encapsulated inside of HTTPS traffic destined for port 443. This is also a slight disadvantage due to the additional traffic overhead of the HTTPS headers, which makes DoH somewhat slower than DoT.

```

Source: https://bartonbytes.com/posts/configure-pi-hole-for-dns-over-tls/

## (Optional) Add a reverse proxy server to use names for local services behind ports

More info: https://www.youtube.com/watch?v=nmE28_BA83w

- Spin up nginx proxy manager (docker compose file included).

- Add an A-record and some CNAME records (optional) in pihole that point to the proxy.

    - WARNING: If nginx proxy manager is running in macvlan, it will be unable to communicate with its host (which is currently your baremetal pihole install), and thus unable to forward requests there (like pihole.local for now) - since pihole admin interface resides on port 80, current workaround for this setup is to simply use A-record for resolving pihole.local.

    - WARNING: Some services (homeassistant, proxmox, etc) require websocket support (enabled in the proxy host's settings - you'll be able to navigate to the login page, but you won't be able to log in otherwise)

- In the proxy, add a 'proxy host' that collates requests for your added A-record / CNAME records and points them all to the proxy's IP address and the port it's listening on (81).

- ...Rinse and repeat for your other services.

## Troubleshooting and post-deployment

- (Windows) To verify pihole's v4 and v6 dns providers have propogated to your device: `ipconfig /all`

- (Windows) Nuclear option to flush your cached dns settings:

    - `ipconfig /release`
    - `ipconfig /renew`
    - `ipconfig /flushdns`
    - `ipconfig /registerDNS` (elevated)
