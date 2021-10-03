# A Raspberry Pi 2 access point tunneling traffic over WireGuard

Steps to setup a WIFI access point using IPv4 and IPv6 support as well as a Wireguard client which tunnels all access point traffic.

Tested on a `Raspberry Pi 2` running *Raspbian GNU/Linux 10 (buster) Kernel 5.10.60-v7+* with a `Ralink Technology, Corp. RT5572 Wireless Adapter` and `eth0` connected to the internet.

------------

###### Remark on the WIFI setup

You might want your WIFI device to run simultaneously as an access point and a WIFI client connecting to the internet. Run `iw phy0 info` and check the *valid interface combinations* section before modifying your setup. For more information check the *udev* folder.

------------

###### Network configuration

This configuration deploys the following network setup

- **Access point**
  IPv4 `192.168.72.1/24`,
  IPv6 `fd72::1/64`
- **WireGuard tunnel**
  IPv4 `10.71.0.1/24` with the Raspberry Pi set to `10.71.0.101`,
  IPv6 `fd71::1/64` with the Pi set to `fd71::101`

All traffic from the `192.168.72.1/24` and the `fd72::1/64` subnets is routed over the WireGuard tunnel.

IPv4 and(!) IPv6 traffic is *masqueraded* on the WireGuard server. In particular, the access point clients do not receive a public IP address without further configuration.

# Preparational steps

Update your Raspberry Pi and install the *wireguard*, *dnsmasq*, and *dnsmasq* packages. You might want to install *tcpdump* and *iperf3* for testing purposes.

    sudo apt-get update && sudo apt-get -y dist-upgrade && sudo apt-get --purge -y autoremove
    sudo apt-get -y install wireguard dnsmasq hostapd tcpdump iperf3


# Wifi access point

In order to setup the access point proceed as follows:
1.  - Copy / merge the content of *hostapd.conf* into `/etc/hostapd/hostapd.conf`.
    - Adjust the `ssid` and the `wpa_passphrase` entries.
1.  - Copy *dnsmasq.conf* into `/etc/dnsmasq.d/` and adjust the `dhcp-range` (IP address ranges) entries if necessary.
1. - Append / merge the content of *dhcpcd.conf.append* to `/etc/dhcpcd.conf` and keep the IP address in line with your `dhcp-range` modifications.
1. - Enable IPv4 and IPv6 forwarding by copying *97-wifi-ap.conf*  to `/etc/sysctl.d/`:

Then edit the `/etc/default/hostapd` file as follows:

    sudo sed -i 's/^#DAEMON_CONF=""$/DAEMON_CONF="\/etc\/hostapd\/hostapd.conf"/g' /etc/default/hostapd

Unmask and enable services on boot

    sudo systemctl unmask hostapd
    sudo systemctl enable hostapd dnsmasq

If the access point is not available after reboot try to debug with the following command

    sudo hostapd -dd /etc/hostapd/hostapd.conf

# WireGuard configuration

On the client side (i.e. the access point) run the following commands:

    sudo sh -c 'umask 077; wg genkey | tee /etc/wireguard/client_ap_priv.key | wg pubkey > /etc/wireguard/client_ap_pub.key'
    sudo sh -c 'umask 077; wg genpsk > /etc/wireguard/preshared.key'
The second command is optional and generates a preshared key in order to add symmetric encryption.

On the server side:

    sudo sh -c 'umask 077; wg genkey | tee /etc/wireguard/server_priv.key | wg pubkey > /etc/wireguard/server_pub.key'

The second command generates a preshared secret in order to use symmetric and asymmetric cryptography in WireGuard.

The generated keys need to be inserted into the `wg0_client.conf` resp. `wg0_server.conf` files and can be deleted afterwards.

- On the Raspberry Pi run
   `sudo cp wg0_client.conf /etc/wireguard/wg0.conf`
   and, if previously modified, change the `Address` entries.
- On the WireGuard server run
   `sudo cp wg0_server.conf /etc/wireguard/wg0.conf`
   and, if previously modified, change the `Address` and the `AllowedIPs` entries.

In order to test the WireGuard configuration run `sudo wg-quick up wg0` on the server and the client. In order to auto-start WireGuard on boot run

    sudo systemctl enable wg-quick@wg0
