---
categories:
- raspberry
date: "2019-03-27T00:00:00Z"
author: "Hugo Alves"
tags:
- raspberry
- vpn
- pihole
- openvpn
title: Install a private OpenVPN server in your home network using a Raspberry Pi
---

In this tutorial you will learn how to install the OpenVPN server on your home network using only a Raspberry Pi.
After completing this guide you will have a fully functional VPN Server in your home.
Using the VPN will protect your internet traffic when you are connected to public or non trusted WiFi networks. While using the VPN your traffic is encrypted and sent directly to your home network, then it will be routed to the destination by your ISP.  
Another advantage is that you can securely access your home network while you are away. For example, you can watch your movies stored in a local NAS, control smart lights, check the security cameras etc... All of these without exposing these services ports over the Internet!

And we will also use a local PiHole as the DNS server. By using PiHole as your DNS sever you can save bandwidth, by eliminating advertisements, block unwanted malware and tracking scripts. You can check [this guide]({% post_url 2019-03-22-pihole-vodafone %}) to see how to install PiHole on the raspberry Pi.

**This is not a method to hide your data from your ISP.** If you want to achieve this you'll need to install the OpenVPN server outside your home network, or buy a VPN service. If you are worried that your ISP is spying you, then this guide is not for you. The objective for this setup is to protect your internet traffic from non trusted networks, like the WiFi that Hotels provide to their customers.

## **Objectives for this tutorial:**

1. Install OpenVpn server on raspberry PI.

2. Use PiHole as your DNS server.

3. Make OpenVpn accessible from outside the home network, i.e., from Internet.

4. Connect to VPN using the OpenVPN client from a IOS device and from Windows 10.

## Requirements

1. **Raspberry Pi**  
In this tutorial I used a Raspberry Pi 1B, it should be enough to run OpenVPN for a single user. But don't expect very high throughputs. With my home connection 200/100 Mbps I could only get 10 Mbps while connected to the VPN.

2. **A static IP or a DDNS**  
If your ISP doesn't provide you a static IP address you will need a DDNS.
A DDNS is just a method of automatically updating a name server. For example the domain "demo-pivpn.duckdns.org" will resolve to my public IP address. But because my public address changes frequently I rely on duckDns service to keep this DNS record constantly updated with my public address.  
If you want a free solution you can use [Duck DNS](https://www.duckdns.org). Check the other tutorial on this site about how you can configure DuckDNS on a Raspberry.

3.  **Port forward configured for the OpenVPN server**  
The Raspberry have a private IP address inside your home network. This address is not exposed on the internet. To make the OpenVPN port available on the Internet you will have to configure a port forward rule on your router.
Basically you will have to forward the traffic that reaches your public IP (on port 1194) to the private IP of the Raspberry, also for the port 1194.
This guide will not explain how to configure this, each router/ISP has its own way of achieving this. For more info  you can check [this page](https://portforward.com/router.htm).

4. **Open Vpn client**  
You must install OpenVpn client on your client device (IOS, Android, windows etc...)


## Install OpenVPN server

To make our task easier we will use [PiVPN](http://www.pivpn.io). PiVPN is a script that provides an easy way to install and configure a OpenVPN server on Raspberry Pi.

Lets connect to Raspberry with SSH and begin the installation with the following commands:

    curl -L https://install.pivpn.io  > pivpn_install.sh
    chmod +x pivpn_install.sh
    ./pivpn_install.sh

The first thing that PiVPN does is to update the system. After that, it downloads OpenVPN Server and launches a wizard to configure OpenVPN server.  
Now you just need to follow the installation wizard to
complete the installation.

##### The following figures explain each step of the wizard:
---

1. Informs that the PiVPN installer will install OpenVPN server

{{< image src="images/step_1.png" alt="pivpn step 1" height="auto">}}

2. To work correctly OpenVPN server needs a static IP address. You have to ensure that the Raspberry Pi is configured with a static IP address. Or that your DHCP server always announces the same IP address for the raspberry.
{{< image src="images/step_2.png" alt="pivpn step 2" height="auto"  >}}

3. Check if the IP address and gateway are correct. If not, cancel the installation and configure the raspberry with the correct IP address.
{{< image src="images/step_3.png" alt="pivpn step 3" height="auto"  >}}

4. Another warning about the importance of having a static IP address. If your network has a DHCP server, ensure that it always gives the same IP address to the raspberry.
{{< image src="images/step_4.png" alt="pivpn step 4" height="auto"  >}}

5. Choose a local user to store the openvpn configurations.
{{< image src="images/step_5.png" alt="pivpn step 5" height="auto"  >}}

6. I will use the default user "pi".
{{< image src="images/step_6.png" alt="pivpn step 6" height="auto"  >}}

7. This is an important warning about keeping your raspberry updated. Since you will have your raspberry exposed to the Internet is is **very** important that you keep it updated.  You should enable unattended Upgrades. With this setting enabled the system will periodic check for security updates and apply them.  
This feature does not eliminate the need of manual updating the raspberry regularly. There are some updates that require a restart and because of that cannot be applied automatically.
{{< image src="images/step_7.png" alt="pivpn step 7" height="auto"  >}}

8. As explained in the previous step, enable unattended Upgrades.
{{< image src="images/step_8.png" alt="pivpn step 8" height="auto"  >}}

9. OpenVPN can either run over UDP or TCP, normally you should use UDP.
You should only choose TCP for specific cases. For example, if the networks where you will be using the VPN block UDP traffic, in that case you can try to run OpenVPN over TCP.
{{< image src="images/step_9.png" alt="pivpn step 9" height="auto"  >}}

10. Choose the port where OpenVPN will listen. I used the default port 1194.
{{< image src="images/step_10.png" alt="pivpn step 10" height="auto"  >}}

11. Confirm the settings.
{{< image src="images/step_11.png" alt="pivpn step 11" height="auto"  >}}

12. OpenVPN 2.4 has introduced some major improvements. Namely the [Elliptic Curve Crypto](https://github.com/OpenVPN/openvpn/blob/master/README.ec)(that is an alternative to RSA keys) and tls-crypt.
Unfortunately these two features are not yet supported on OpenVPN client for IOS. So if you plan to use an IOS device then don't activate this feature. The remaining of the guide is based on having this setting disabled, there are some different steps if you choose activate 2.4 features.
{{< image src="images/step_12.png" alt="pivpn step 12" height="auto"  >}}

13. Select the size for the server RSA key, 2048 is a good balance between speed and security.
{{< image src="images/step_13.png" alt="pivpn step 13" height="auto"  >}}

14. This step will generate the server RSA key pair and the [Diffie-Hellman](https://en.wikipedia.org/wiki/Diffie–Hellman_key_exchange) parameters. This step will take a lot of time on Raspberry Pi 1B... so be patient.
To avoid aborting the process due to PC sleep or network problems you should use a terminal multiplexer like [tmux](https://github.com/tmux/tmux/wiki).
{{< image src="images/step_14.png" alt="pivpn step 14" height="auto"  >}}
{{< image src="images/step_14_DH.png" alt="pivpn step 14 result" height="auto"  >}}


15. Here you will have to choose how you will connect to the OpenVPN server. You can either use your public IP address or a domain Name. I do not recommend using the public IP address because it will probably change when your ISP renews the DHCP lease.
You should configure a DDNS, you can configure one at [DuckDNS](https://www.duckdns.org), check the DuckDNS tutorial on this blog.
{{< image src="images/step_15.png" alt="pivpn step 15" height="auto"  >}}

16. I've previously configured this domain at duckdns.org. This domain name will resolve to your public IP address.
{{< image src="images/step_16.png" alt="pivpn step 16" height="auto"  >}}

17. Just confirm the domain you introduced.
{{< image src="images/step_17.png" alt="pivpn step 17" height="auto"  >}}


18. In this step you can configure the DNS server that will be used by your devices. We have two options here, either we use a public DNS server, like Google or Cloudflare, or we use a custom one.  
We will use the last option. Our DNS server will be a PiHole running on the local network (in my case it is also on the raspberry). For information about how to install PiHole check [this blog post]({% post_url 2019-03-22-pihole-vodafone %}).
{{< image src="images/step_18_dns_custom_1.png" alt="pivpn step 18" height="auto"  >}}

19. Since we have chosen to use PiHole as the DNS server we need to fill its IP address in this step. This is the local address, it doesn't need to be a public address.
{{< image src="images/step_18_dns_custom_2.png" alt="pivpn step 18 2" height="auto"  >}}


20. Make sure that the DNS server 1 field is the private IP address of the machine running the PiHole.
{{< image src="images/step_19.png" alt="pivpn step 19" height="auto"  >}}

21. Last step! Reboot the raspberry!
{{< image src="images/step_21.png" alt="pivpn step 20" height="auto"  >}}


## Generate a client certificate

The OpenVPN server is installed and ready to accept connections. But before that, we need to generate the client certificates for each client that will use the VPN.

The client certificate is an RSA keypair that will be used to authenticate and secure the communication between your device and OpenVPN server.
You will need the client certificate in order to connect to the OpenVPN server. You can see this certificate as your username and password.

The client generations is very easy, you just need to run the following command: `pivpn add`

{{< image src="images/pivpn_add.png" alt="pivpn add client" height="auto"  >}}

To create a client you just need to follow the instructions. You have to give a username and a password to encrypt the private key.

This process will generate the `/home/pi/ovpns/my-vpn-client.ovpn` file:

    client
    dev tun
    proto udp
    remote demo-pivpn.duckdns.org 1194
    resolv-retry infinite
    nobind
    persist-key
    persist-tun
    key-direction 1
    remote-cert-tls server
    tls-version-min 1.2
    verify-x509-name server_66iVDlc1xRPTTOSa name
    cipher AES-256-CBC
    auth SHA256
    auth-nocache
    verb 3
    <ca>
    -----BEGIN CERTIFICATE-----
    {A private key}
    -----END CERTIFICATE-----
    </ca>
    <cert>
    -----BEGIN CERTIFICATE-----
    {The certificate}
    -----END CERTIFICATE-----
    </cert>
    <key>
    -----BEGIN ENCRYPTED PRIVATE KEY-----
    {A private key}
    -----END ENCRYPTED PRIVATE KEY-----
    </key>
    <tls-auth>
    #
    # 2048 bit OpenVPN static key
    #
    -----BEGIN OpenVPN Static key V1-----
    {server key}
    -----END OpenVPN Static key V1-----
    </tls-auth>

You can use this file with OpenVPN client app to connect to your VPN.
Now you must copy this file to the device from where you will connect to the VPN, for example your laptop or smartphone. You can copy the file with FTP or SFTP. Or, since this file is just text, you can also copy the content of the file and paste it in a new file(my-vpn-client.ovpn) on your device.

If you have more than one device you should generate one client for each. Do not reuse the .ovpn configuration in multiple devices.

## Connect to the VPN

To connect to the VPN you will need a VPN client, there are multiples alternatives.
For Windows, IOS and Android you can use the [Open VPN client](https://openvpn.net/community-downloads/).
For MacOs you can use [Tunnelblick](https://tunnelblick.net/). Despite the one you choose the only thing you will need is the .ovpn file that you generated in the previous step.

#### OpenVPN client for IOS

- Download OpenVPN client from app store: https://itunes.apple.com/pt/app/openvpn-connect/id590379981?mt=8
- Copy the .ovpn file previously generated to Icloud/Dropbox/Drive etc...
- Navigate to the file and open with OpenVPN client
- Enter the password chosen during the client creation step
- And click connect, you are now connected!

While connected to the VPN you can see a "VPN" symbol on the top left corner, right next to the WiFi symbol.

{{< image src="images/ios_openvpn_install.gif" alt="openvpn client windows " height="auto"  >}}


#### OpenVPN client for Windows

- Download OpenVPN client from https://openvpn.net/community-downloads/
- Copy the .ovpn file previously generated to the machine
- Right click on OpenVPN client icon on the notification area (near the clock). And then "Import File"
- Select the .ovpn file
- Go to the icon on the notification bar and right click it. Then click "Connect"
- Enter the password chosen during the client creation step
- And click ok, you are now connected!

{{< image src="images/windows_openvpn_install.gif" alt="openvpn client windows " height="auto"  >}}

## Results

As said before you will not get the full throughput that you normal get in your home. Your speed will be limited by multiple factors:
- You home network Upload speed
- Raspberry Pi processing capacity
- Visited network speed

In this case my home network is 200/100 Mbps and I tested the VPN from a WiFi network that also provides around 200/100 Mbps.  
With this setup I could get around 10/10 Mbps running from the Raspberry Pi 1B, this is one of the oldest Raspberry's, with a 700 MHz cpu and 512 MB of ram.

Running the same test with a Raspberry Pi 3B+ I got no noticeable improvements on the speedtest

{{< image src="images/speedtest_vpn_3b.png" alt="pivpn speedtest " height="auto"  >}}.

The 10/10 Mbps provided by Raspberry Pi 1B is enough to my daily browsing outside home. It was also enough to stream some series from my NAS transcoded to a Iphone screen.
And the biggest advantage is the [power consumption](https://www.jeffgeerling.com/blogs/jeff-geerling/raspberry-pi-zero-power) of the Raspberry Pi 1B, consuming as low as 0.4W.
