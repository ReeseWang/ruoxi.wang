# 用`isc-dhcp-server`引导树莓派网络启动

树莓派3B、3B+、2B v1.2和4（需要[修改Bootloader](https://hackaday.com/2019/11/11/network-booting-the-pi-4/)）都支持从有线网网络启动，网络启动可以使一些型号的树莓派无需SD卡即可使用。用`dnsmasq`引导树莓派启动的方法在[官方文档](https://www.raspberrypi.org/documentation/hardware/raspberrypi/bootmodes/net_tutorial.md)中已经说得很清楚了，但如果内网已经配置成用[ISC的DHCP服务端](https://www.isc.org/dhcp/)给客户机分配IP地址，我们如何把它配置成能引导树莓派网络启动的程序呢？

通过对`dnsmasq`的DHCP响应抓包分析可得，在`/etc/dhcp/dhcpd.conf`中添加以下配置即可使`isc-dhcp-server`发出同样的DHCP响应：

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
  option RPi.menu-item "Raspberry Pi Boot ";

  option routers <网关IP>;
  next-server <TFTP服务器IP>;

  host rpi-1 {
    hardware ethernet b8:27:eb:xx:xx:xx;
    fixed-address 192.168.1.140;
  }
}
...
```

与此同时，我们仍然需要`dnsmasq`提供TFTP服务。但是要禁用`dnsmasq`的DHCP服务，不然`dnsmasq`会因为`isc-dhcp-server`已经在监听UDP 53端口而不能启动。

`/etc/dnsmasq.conf`:

```
port=0
enable-tftp
tftp-root=/srv/raspberrypi/boot
interface=eth1
```