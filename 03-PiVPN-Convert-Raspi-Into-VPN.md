# Convert your Raspberry into a VPN
If you want to be able to access your homelab from the outside of your network without the need of opening your network to the world and dealing with all the security issues that this means.

Turning your Raspberry into a VPN is pretty easy thanks to the folks developing [PiVPN](https://www.pivpn.io/). The simplest way to setup and manage a VPN, designed for Raspberry Pi.  But as everything in these docs, it also works with any kind of server like Ubuntu Server.

If you are running a Raspberry, it is recommended to run Raspbian Lite.

I had an old Raspberry Pi 3b+ at home which was unused for ages. I will be using this RPI3 as my PiVPN and I will also setup a second PiHole as a backup of the main one.

To install [PiVPN](https://www.pivpn.io/) you just need to run:

- `curl -L https://install.pivpn.io | bash`

This will guide you though a step-by-step process which will help you configure your PiVPN. I strongly recommend you to use Wireguard as it is **much** more efficient than OpenVPN.

Once you have set up your PiVPN don't forget to forward the port you set up (default is `51820`) from your router to your PiVPN local IP. You can do this from your router configuration, probably under a section called `Port Forwarding`.

Your configuration should look like somtething like this:

| Name | Type | Port Start | Port End | NAT interface | Private IP | Private Port |
| --- | --- | --- | --- | --- | --- | --- |
| Wireguard | Port-Range | 51820 | 51820 | TPC or UDP (Both) | eth0.vxxx | 192.168.1.223 |
