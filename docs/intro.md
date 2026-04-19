TWRP recovery (or any other modern recovery) is an ideal environment for benchmarking and generally booting Linux distributions. It includes all the manufacturer's binaries and generally runs stably. In this notebook, I'll explain how to install Ubuntu on TWRP.

setup_ubuntu_rootfs.md is a note that explains in more detail how to create your own rootfs. My Ubuntu installation can be downloaded [here](https://drive.google.com/file/d/1heMCehAZoDS_dTN4GH2TcRQuno63to9P/view?usp=sharing).

The directory where I'll unpack the Ubuntu rootfs will be /data/ubuntu. There's no need to format the userdata partition. Not all folders have File Based Encryption (FDE).

First, put the Ubuntu rootfs on a RAM disk, and then unpack it onto Ubuntu's internal storage. TWRP uses so little RAM that even if I run the Linux kernel build on 8 cores, I'll still have 3GB of free RAM, without swap (or zRAM). But we won't unpack Ubuntu onto a ramdisk. We'll install it permanently. I think TWRP has a time limit(10h or so).

```
adb push ubuntu_rootfs_shared.tar /tmp
mkdir -p /data/ubuntu; cd /data/ubuntu
tar xvf /tmp/ubuntu_rootfs_shared.tar
```

Next, we bind-mount to access various Linux pseudo-filesystems.

```
cd /data/ubuntu
for i in dev proc sys tmp config; do mount -o rbind /$i $i/; done
```

Now we chroot Ubuntu. Note that without updating the PATH, the shell can't find any programs. We set the governor to performance to ensure all cores run at maximum speed.

```
echo performance | tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
chroot .
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin:$PATH
alias sudo=''
bash
```

**Networking**

Theoretically, it's possible to set up Wi-Fi, but you need to load the Wi-Fi modules and they must be compiled with the kernel. I plan to upload a ready-made TWRP with Wi-Fi in the future. Now we'll set up internet via USB gadget. The host must be running Linux.


TWRP Device commands:
(inside ubuntu)

```
#enable rndis
cd /config/usb_gadget/g1
mkdir functions/rndis.usb0
echo "02:00:00:00:00:01" > functions/rndis.usb0/dev_addr
echo "02:00:00:00:00:02" > functions/rndis.usb0/host_addr
ln -s functions/rndis.usb0 configs/b.1/
#disconnect to enforce!

busybox ifconfig rndis0 10.0.0.2 netmask 255.0.0.0 up
#Set default gateway!!
busybox ip route add default via 10.0.0.1 dev rndis0
echo "nameserver 8.8.8.8" > /etc/resolv.conf
echo "nameserver 8.8.4.4" >> /etc/resolv.conf
busybox ping oko.press
```

Host commands:
```
#customize to yourself
SRC=wlp3s0
DEST=enx020000000002

sudo iptables -t nat -A POSTROUTING -o $SRC -j MASQUERADE
sudo iptables -A FORWARD -i $SRC -o $DEST -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -i $DEST -o $SRC -j ACCEPT

#Set IP
sudo busybox ifconfig $DEST 10.0.0.1 netmask 255.0.0.0 up
sudo sysctl -w net.ipv4.ip_forward=1
```

**Benchmark**

Modern mobile SoCs have the big.LITTLE architecture, which means they have 4 energy-efficient cores (cluster) and 4 high-performance cores. To properly benchmark, you need to take this into account, for example, by assigning specific cores to the benchmark. Test "big" and "little" separately. The Cortex A53 is an in-order core: instructions execute in order, the core size on a piece of silicon is tiny, and the architecture inside the core is simpler. All modern x86 cores are out-of-order, except for the Intel Atom n270 – which is therefore TERRIBLY slow and practically a meme. Much has changed now, and I believe that Intel's latest CPUs also use the big.LITTLE architecture, and therefore also in-order.

This budget CPU doesn't use out-of-order, but you can identify the perfomance cores by their maximum frequency or core "capacity."

```
for cpu in /sys/devices/system/cpu/cpu*/cpu_capacity; do
echo "$cpu: $(cat $cpu)"
done

for cpu in /sys/devices/system/cpu/cpu*/cpufreq/cpuinfo_max_freq; do
echo "$cpu: $(cat $cpu)"
done

```
