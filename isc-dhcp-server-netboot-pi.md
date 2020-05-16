# Configure `isc-dhcp-server` to Network Boot Your Raspberry Pi

Raspberry Pi 3B, 3B+, 2B v1.2 and 4 ([bootloader modification needed](https://hackaday.com/2019/11/11/network-booting-the-pi-4/)) are able to boot from wired network, thus possibly eliminates the need for an SD card. The process of setting up `dnsmasq` to provide DHCP service to network boot a Raspberry Pi is already documented [here](https://www.raspberrypi.org/documentation/hardware/raspberrypi/bootmodes/net_tutorial.md). But when your LAN is already using [DHCP server program from ISC](https://www.isc.org/dhcp/) to allocate IPs for hosts, how do you configure it to boot a Raspberry Pi as `dnsmasq` does?

By capturing the DHCP packets from `dnsmasq`, I could determine the configuration that must to be added to `/etc/dhcp/dhcpd.conf` to achieve the same DHCP reply:

```
...
option space RPi code width 1 length width 1;
option RPi.discovery code 6 = unsigned integer 8;
option RPi.menu-prompt code 10 = text;
option RPi.menu-item code 9 = text;
group {
  vendor-option-space RPi;
  option RPi.discovery 3;
  option RPi.menu-prompt "PXE";
  option RPi.menu-item "Raspberry Pi Boot";

  option routers <IP of the gateway>;
  next-server <IP of the TFTP server>;

  host rpi-1 {
    hardware ethernet b8:27:eb:xx:xx:xx;
    fixed-address 192.168.1.140;
  }
}
...
```

 However, `dnsmasq` still need to be running to provide TFTP service. It just needs to have DHCP functionality disabled or it will fail to start due to `isc-dhcp-server` already listening on UDP port 53:

`/etc/dnsmasq.conf`:

```
port=0
enable-tftp
tftp-root=/srv/raspberrypi/boot
interface=eth1
```

