# Set up the RPI4 server

Download Ubuntu Server image or Raspbian for Raspberry Pi. It is recommended to get the LTS version.

- Ubuntu Server: https://ubuntu.com/download/raspberry-pi
- Raspbian: https://www.raspberrypi.org/software/operating-systems/

Raspbian is more lightweight and more suitable for Raspberry.

Burn the image on the SD card using [**Balena Etcher**](https://www.balena.io/etcher/) (or [Rufus](https://rufus.ie/en_US/)).

Insert the SD card to the Raspberry and connect your Raspberry to a power supply so it will boot. You should connect your Raspberry to some kind of display using the HDMI ports for the first configurations (at least until SSH is ready).

Once booted, log in using the default credentials for your server:

- User: `ubuntu` Password: `ubuntu` if we are on **Ubuntu Server**.
- User: `pi` Password: `raspberry` if we are on **Raspbian**.

Once logged in, change the password. It should automatically ask you to change it. If not, use `passwd` to change it.

Depending on what OS you chose, you can jump to:
- [Raspbian config](https://github.com/pluja/Selfhosted-Google-Photos-Alternative/blob/main/01-Setting-Up-The-Server.md#raspbian-config)
- [Ubuntu config](https://github.com/pluja/Selfhosted-Google-Photos-Alternative/blob/main/01-Setting-Up-The-Server.md#ubuntu-config)

## Raspbian config

> The `sudo raspi-config` command comes very handy for basic configuration. Use it to enable SSH server and change keyboard layout before proceding.

Now we will configure the internet connections for the RPI. We will be setting up the Wi-Fi connection with DHCP4 (will assign an IP dynamically; this is for the first updates and so) and then we will be configuring the Ethernet port with a static IP for when we set up the RPI as a server.

To setup the WiFi connection use `sudo raspi-config` and search through the menus for the WiFi configuration. There, enter the SSID (network name) and Password to connect to your network. Once done, reboot the RPI.

Now update all packages with:
`sudo apt-get update && sudo apt-get upgrade`

Now let's configure the static IP for the `eth0` port.

For this we will need to edit the `/etc/dhcpcd.conf` file:
- `sudo nano /etc/dhcpcd.conf`

Add the following lines at the end of the file:

```
interface eth0
static ip_address=192.168.1.222/24    
static routers=192.168.1.1
static domain_name_servers=192.168.1.1 9.9.9.9
```

This way we are assigning the static ip `192.168.1.222` to the raspberry pi `eth0` port. Also we are using `9.9.9.9` as the DNS provider (Quad9). You can change this to suit your needs.

Now shut down the RPI with `shutdown now` and connect the Ethernet cable. Once booted, try to SSH to the RPI-4 with the assigned IP:

`ssh pi@192.168.1.222`

Once you have `ssh` to the RPI, you can proceed to secure the SSH configuration. To do so we will deactivate the password and root login for SSH for security.

To generate a private+public key pair on *NIX systems, launch this command on the local machine (your workstation, not RPI):

`ssh-keygen -t rsa`

You can use the default folder destination for the keys. Then you will need to copy the generated keys to the remote machine (RPI):

`ssh-copy-id -i $HOME/.ssh/id_rsa.pub pi@192.168.1.222`

Now you can try to `ssh` again to the RPI and see that you are no longer prompted for a password:

`ssh pi@192.168.1.222`

> NOTE: make sure to back up your ssh keys (`$HOME/.ssh/` directory)

Now we will edit the SSH config to make it more secure:

`sudo nano /etc/ssh/sshd_config`

You will need to find the following variables and set the values to the ones you can see here:

```
ChallengeResponseAuthentication no
PasswordAuthentication no
UsePAM no
PermitRootLogin no
```

Once done, restart the SSH service:

`sudo systemctl reload ssh`

And finally, install a basic firewall protection:

`sudo apt install fail2ban`

You are done with setting up your server. Now you can proceed to the second step of the guide.

## Ubuntu config

Now we will configure the internet connections for the RPI. We will be setting up the Wi-Fi connection with DHCP4 (will assign an IP dynamically; this is for the first updates and so) and then we will be configuring the Ethernet port with a static IP for when we set up the RPI-4 as a server.

To do so we will need to edit the file `50-cloud-init.yaml`

- `sudo vim /etc/netplan/50-cloud-init.yaml`

Once we are editing it we will paste this configuration (replacing whatever is in the file):

```
network:
    version: 2
    ethernets:
        eth0:
          dhcp4: no
          addresses:
            - 192.168.1.221/24
          gateway4: 192.168.1.1
          nameservers:
            addresses: [9.9.9.9, 1.1.1.1]
    wifis:
        wlan0:
            optional: true
            access-points:
                "<SSID>":
                     password: "<YourW1f1PassW0rD"
            dhcp4: true
```

Now we will save using `ESC` and then the command `:wq` to exit and save Vim. Make sure to change anything that is between `<>` to suit your network. Also check that the `gateway4` (router) is the same for your network. And assign the IP of your choice: with this config the RPI-4 IP will be `192.168.1.221`.

Once done, apply this config with `sudo netplan apply`.

Now we will configure SSH so we can control the RPI from another computer; and we will also deactivate the password and root login for SSH for security.

To generate a private+public key pair on *NIX systems, launch this command on the local machine (not RPI):

`ssh-keygen -t rsa`

You can use the default folder destination for the keys. Then you will need to copy the generated keys to the remote machine (RPI):

`ssh-copy-id -i $HOME/.ssh/id_rsa.pub ubuntu@192.168.1.221`

Now you are ready to connect via SSH to the RPI:

`ssh ubuntu@192.168.1.221`

Now we will edit the SSH config to make it more secure:

`sudo vim /etc/ssh/sshd_config`

You will need to find the following variables and set the values to the ones you can see here:

```
ChallengeResponseAuthentication no
PasswordAuthentication no
UsePAM no
PermitRootLogin no
```

Once done, restart the ssh service:

`sudo systemctl reload ssh`

And finally, install a basic firewall protection:

`sudo apt install fail2ban`

You are done with setting up your server. Now you can proceed to the second step of the guide.
