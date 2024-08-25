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

    - (In router) Configure pihole as _SOLE_ v4 dhcp / dns server on your network.

    - (In router) Configure pihole as _SOLE_ v6 dhcp / dns server on your network (requires auto config to be enabled on same advanced > v6 settings page).

- (Optional) Add firebog lists: `firebog.net`

    - NOTE: The lists with green checkmarks are generally safe and won't break websites.

    - NOTE: To get them in copy/paste format, navigate to the csv version link.

    - NOTE: To activate your changes, run `pihole -g` (as per web gui) and restart dns resolver in settings.

## (Optional) Configure Pi Hole for DNS Over TLS

Source: https://bartonbytes.com/posts/configure-pi-hole-for-dns-over-tls/

- Run: `sudo apt install -y unbound dnsutils`

- There are now two dns providers on your pihole. Fix this by grabbing a config file which will tell unbound to use port 5533: `sudo wget https://bartonbytes.com/pihole.txt -O /etc/unbound/unbound.conf.d/pihole.conf`

- Run: `sudo systemctl restart unbound && sudo systemctl enable unbound`

- Verify: `dig @127.0.0.1 example.com -p 5533`

- Now, we need to tell Pi-hole’s dnsmasq to use this local port as it’s upstream DNS server. In the GUI, disable the upstream dns provider you selected during install and go to Settings -> DNS, and set a custom IPv4 server with the value `127.0.0.1#5533` (tick its checkbox and click save).

- Now we must restart Pi-hole: `sudo systemctl restart pihole-FTL`

## Troubleshooting and post-deployment

- (Windows) Nuclear option to flush your cached dns settings:

    - 
