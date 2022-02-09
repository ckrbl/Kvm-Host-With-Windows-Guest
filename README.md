# KVM Host with Windows Guest and GPU passthrough

This is a guide based on [GreyWolfTech's YouTube video](https://www.youtube.com/watch?v=dsDUtzMkxFk) that had to be adapted to my particular hardware to work.

List of hardware used:
 - Intel i7-7700K
 - ASUS STRIX Z270F Gaming, UEFI mode
 - 32 GiB RAM
 - Intel HD Graphics 630
 - NVIDIA GeForce GTX 1080 Ti

## Deployment Specific Modifications From Original Guide

### Integrated Graphics Notes

Had to install `intel-microcode xorg-intel-driver`

And then do `apt-get install --reinstall xorg-input`

### For the Guest

I used i440FX chipset.

Then after do virsh edit <vm_name>

At the start add:
`<domain type='kvm' xmlns:qemu='http://libvirt.org/schemas/domain/qemu/1.0'>`

In the middle add:
```
    <hyperv>
      <relaxed state='off'/>
      <vapic state='off'/>
      <spinlocks state='off'/>
    </hyperv>
    <kvm>
      <hidden state='on'/>
    </kvm>
    <vmport state='off'/>
```

At the end before the last line add:
```
  <qemu:commandline>
    <qemu:arg value='-cpu'/>
    <qemu:arg value='host,hv_time,kvm=off,hv_vendor_id=null'/>
  </qemu:commandline>
```

### USB Passthrough Bug

Edit `/etc/apparmor.d/abstractions/libvirt-qemu`. Change/add:

```
  # for usb access
  /dev/bus/usb/ rw,

  /dev/bus/usb/*/[0-9]* rw,
  /run/udev/** rw,
  /etc/udev/udev.conf r,
  /sys/bus/ rw,
  /sys/class/ rw,
```

restart apparmor, libvirtd

Disable ROM BAR for the USB3.1 passthrough.

## Port Forwarding

```
iptables -t nat -I PREROUTING -p udp --dport 27020 -j DNAT --to 192.168.122.33:27020
iptables -t nat -I PREROUTING -p udp --dport 25565 -j DNAT --to 192.168.122.33:25565
iptables -t nat -I PREROUTING -p tcp --dport 25565 -j DNAT --to 192.168.122.33:25565
iptables -t nat -A POSTROUTING -s 192.168.122.0/24 -j MASQUERADE
iptables -I FORWARD -o virbr0 -d 192.168.122.33 -j ACCEPT
```

## Bridged Networking

```
brctl addbr br0
brctl addif enp0s31f6
ifconfig enp0s31f6 0
dhclient br0
```

Next, open the NIC settings in virt-manager and choose Network Source: Bridge host br0, device model virtio.

Apply and reboot

## How to Resize Windows's Hard Drive

```
cd /opt/kvm/
qemu-img resize WinHDD.qcow2 +150G
```

## How to Fix Windows Updates Requiring a Reboot Getting Stuck in "Paused"

Let it shut itself down.

Click Resume (fails), then Reboot, Then Shut Down, then Force Reset, then UNPAUSE using the button at the top. Windows should resume as through a reboot occurred, and the update should successfully be complete.

## How to Fix Video Thrashing and Game Stuttering

```
sudo cpupower frequency-set -g performance
```
